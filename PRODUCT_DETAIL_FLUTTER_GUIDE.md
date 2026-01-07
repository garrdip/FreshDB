# 제품 상세 화면 Flutter 구현 가이드

> 클라이언트용 Flutter 앱에서 제품 상세 화면을 구현할 때 참고하는 문서입니다.

## 1. 화면 구조 개요

```
┌─────────────────────────────────────┐
│          상단 헤더 (AppBar)           │
│  ← 뒤로가기          공유 / 찜 버튼    │
├─────────────────────────────────────┤
│                                     │
│         제품 이미지 슬라이더            │
│        (PageView + Indicator)        │
│                                     │
├─────────────────────────────────────┤
│  [추천] [인기] [갑각류]  ← 뱃지들       │
│  킹크랩 (살)           ← 제품명        │
│  890,000원  1,120,000원 20%OFF       │
├─────────────────────────────────────┤
│  제품 상태: [활어(Fresh Frozen)] 추천   │
├─────────────────────────────────────┤
│  크기 선택:                           │
│  ┌─────┐ ┌─────┐ ┌─────┐            │
│  │  중  │ │  대  │ │ 특대 │            │
│  │Medium│ │Large │ │  XL  │            │
│  │2.5kg │ │ 3kg  │ │ 4kg  │            │
│  │      │ │+5만원│ │+15만원│            │
│  └─────┘ └─────┘ └─────┘            │
├─────────────────────────────────────┤
│  수량 선택: [-] 2 [+]                 │
├─────────────────────────────────────┤
│  [상품정보] [리뷰] [배송]  ← 탭바       │
├─────────────────────────────────────┤
│  상품 설명 / 특징 / 포함 항목          │
├─────────────────────────────────────┤
│                                     │
│  ┌─────────────────────────────┐    │
│  │      장바구니 담기 버튼        │    │
│  └─────────────────────────────┘    │
│  [홈] [장바구니] [주문] [마이]         │
└─────────────────────────────────────┘
```

## 2. Supabase 데이터 스키마

### 2.1 products 테이블

| 컬럼명 | 타입 | 필수 | 설명 |
|--------|------|------|------|
| `id` | uuid | O | 제품 고유 ID (PK) |
| `sku` | varchar | O | 제품 SKU 코드 |
| `name` | varchar | O | 제품명 |
| `description` | text | - | 제품 설명 |
| `category` | varchar | O | 카테고리 (FISH, SHELLFISH, CRUSTACEAN, MOLLUSK) |
| `price` | numeric | O | 판매가 |
| `original_price` | numeric | - | 정가 (할인 전 가격) |
| `freshness_grade` | varchar | O | 선도등급 (sashimi, premium, grade_a, grade_b, grade_c, processing) |
| `storage_type` | varchar | O | 보관/상태 타입 |
| `origin` | varchar | - | 원산지 |
| `company` | varchar | - | 공급사 |
| `unit` | varchar | O | 판매 단위 (kg, g, piece, box) |
| `min_quantity` | numeric | - | 최소 주문 수량 |
| `stock_quantity` | numeric | - | 재고 수량 |
| `image_urls` | text[] | - | 이미지 URL 배열 |
| `is_flash_sale` | boolean | - | 타임세일 여부 |
| `flash_sale_start` | timestamptz | - | 타임세일 시작 시간 |
| `flash_sale_end` | timestamptz | - | 타임세일 종료 시간 |
| `is_popular` | boolean | - | 인기상품 여부 |
| `is_featured` | boolean | - | 추천상품 여부 |
| `status` | varchar | - | 상태 (active, inactive, out_of_stock, deleted) |
| `weight` | numeric | - | 기본 중량 |
| `weight_unit` | varchar | - | 중량 단위 |
| `is_preorder` | boolean | - | 예약주문 여부 |
| `delivery_type` | varchar | - | 배송 유형 (standard, express, live, same_day) |
| `free_shipping` | boolean | - | 무료배송 여부 |
| `free_shipping_threshold` | numeric | - | 무료배송 기준 금액 |
| `metadata` | jsonb | - | 추가 메타데이터 (JSON) |

### 2.2 product_variants 테이블

| 컬럼명 | 타입 | 필수 | 설명 |
|--------|------|------|------|
| `id` | uuid | O | 옵션 고유 ID (PK) |
| `product_id` | uuid | O | 제품 ID (FK) |
| `variant_type` | varchar | O | 옵션 타입 (size, condition) |
| `size_code` | varchar | - | 크기 코드 (MEDIUM, LARGE, XLARGE) |
| `name` | varchar | O | 옵션 한글명 |
| `name_en` | varchar | - | 옵션 영문명 |
| `value` | varchar | - | 상세값 (예: "2.5kg/마리") |
| `price_adjustment` | numeric | - | 가격 조정 (+/- 금액) |
| `stock_quantity` | numeric | - | 해당 옵션 재고 |
| `is_default` | boolean | - | 기본 선택 여부 |
| `is_recommended` | boolean | - | 추천 옵션 여부 |
| `badge_text` | varchar | - | 뱃지 텍스트 (예: "추천", "BEST") |
| `display_order` | integer | - | 표시 순서 |
| `is_active` | boolean | - | 활성화 여부 |

### 2.3 metadata JSON 구조

```json
{
  "features": [
    "최상급 M.S. 킹크랩 54%",
    "홍연어 수입수 최신 배송",
    "산지 직배송으로 최상 선도",
    "당일손질 당일발송 원칙 준수"
  ],
  "inclusions": [
    "Pkg 10kg 박스 1",
    "4-6마리 정도",
    "신선도 보장 인증서",
    "기프트 카드 1장"
  ],
  "urgent_notice": {
    "title": "도매 긴급 안내",
    "items": ["이번주 수량한정", "빠른 품절 예상"]
  },
  "promo_text": "20% 할인 중",
  "delivery_info": {
    "estimated_days": 1,
    "guarantee": "익일 도착 보장"
  }
}
```

## 3. Supabase 쿼리 예시

### 3.1 제품 상세 조회

```dart
// 제품 기본 정보 조회
final productResponse = await supabase
    .from('products')
    .select('*')
    .eq('id', productId)
    .eq('status', 'active')
    .single();

// 제품 옵션(variants) 조회
final variantsResponse = await supabase
    .from('product_variants')
    .select('*')
    .eq('product_id', productId)
    .eq('is_active', true)
    .order('variant_type')
    .order('display_order');
```

### 3.2 조인 쿼리 (한 번에 조회)

```dart
final response = await supabase
    .from('products')
    .select('''
      *,
      product_variants!inner(*)
    ''')
    .eq('id', productId)
    .eq('status', 'active')
    .eq('product_variants.is_active', true);
```

## 4. Flutter 모델 클래스

### 4.1 Product 모델

```dart
class Product {
  final String id;
  final String sku;
  final String name;
  final String? description;
  final String category;
  final double price;
  final double? originalPrice;
  final String freshnessGrade;
  final String storageType;
  final String? origin;
  final String? company;
  final String unit;
  final int minQuantity;
  final int stockQuantity;
  final List<String>? imageUrls;
  final bool isFlashSale;
  final DateTime? flashSaleStart;
  final DateTime? flashSaleEnd;
  final bool isPopular;
  final bool isFeatured;
  final String status;
  final double? weight;
  final String? weightUnit;
  final bool isPreorder;
  final String? deliveryType;
  final bool freeShipping;
  final double? freeShippingThreshold;
  final ProductMetadata? metadata;

  // 할인율 계산
  int get discountPercent {
    if (originalPrice == null || originalPrice! <= price) return 0;
    return ((1 - price / originalPrice!) * 100).round();
  }

  // 카테고리 한글명
  String get categoryLabel {
    const categories = {
      'FISH': '어류',
      'SHELLFISH': '조개류',
      'CRUSTACEAN': '갑각류',
      'MOLLUSK': '연체류',
    };
    return categories[category] ?? category;
  }

  // 선도등급 한글명
  String get freshnessGradeLabel {
    const grades = {
      'sashimi': '회급',
      'premium': '프리미엄',
      'grade_a': 'A급',
      'grade_b': 'B급',
      'grade_c': 'C급',
      'processing': '가공용',
    };
    return grades[freshnessGrade] ?? freshnessGrade;
  }

  factory Product.fromJson(Map<String, dynamic> json) {
    return Product(
      id: json['id'],
      sku: json['sku'],
      name: json['name'],
      description: json['description'],
      category: json['category'],
      price: (json['price'] as num).toDouble(),
      originalPrice: json['original_price'] != null
          ? (json['original_price'] as num).toDouble()
          : null,
      freshnessGrade: json['freshness_grade'],
      storageType: json['storage_type'],
      origin: json['origin'],
      company: json['company'],
      unit: json['unit'],
      minQuantity: (json['min_quantity'] as num?)?.toInt() ?? 1,
      stockQuantity: (json['stock_quantity'] as num?)?.toInt() ?? 0,
      imageUrls: (json['image_urls'] as List?)?.cast<String>(),
      isFlashSale: json['is_flash_sale'] ?? false,
      flashSaleStart: json['flash_sale_start'] != null
          ? DateTime.parse(json['flash_sale_start'])
          : null,
      flashSaleEnd: json['flash_sale_end'] != null
          ? DateTime.parse(json['flash_sale_end'])
          : null,
      isPopular: json['is_popular'] ?? false,
      isFeatured: json['is_featured'] ?? false,
      status: json['status'] ?? 'active',
      weight: json['weight'] != null
          ? (json['weight'] as num).toDouble()
          : null,
      weightUnit: json['weight_unit'],
      isPreorder: json['is_preorder'] ?? false,
      deliveryType: json['delivery_type'],
      freeShipping: json['free_shipping'] ?? false,
      freeShippingThreshold: json['free_shipping_threshold'] != null
          ? (json['free_shipping_threshold'] as num).toDouble()
          : null,
      metadata: json['metadata'] != null
          ? ProductMetadata.fromJson(json['metadata'])
          : null,
    );
  }
}
```

### 4.2 ProductVariant 모델

```dart
class ProductVariant {
  final String id;
  final String productId;
  final String variantType; // 'size' | 'condition'
  final String? sizeCode;
  final String name;
  final String? nameEn;
  final String? value;
  final double priceAdjustment;
  final int? stockQuantity;
  final bool isDefault;
  final bool isRecommended;
  final String? badgeText;
  final int displayOrder;
  final bool isActive;

  factory ProductVariant.fromJson(Map<String, dynamic> json) {
    return ProductVariant(
      id: json['id'],
      productId: json['product_id'],
      variantType: json['variant_type'],
      sizeCode: json['size_code'],
      name: json['name'],
      nameEn: json['name_en'],
      value: json['value'],
      priceAdjustment: (json['price_adjustment'] as num?)?.toDouble() ?? 0,
      stockQuantity: (json['stock_quantity'] as num?)?.toInt(),
      isDefault: json['is_default'] ?? false,
      isRecommended: json['is_recommended'] ?? false,
      badgeText: json['badge_text'],
      displayOrder: (json['display_order'] as num?)?.toInt() ?? 0,
      isActive: json['is_active'] ?? true,
    );
  }
}
```

### 4.3 ProductMetadata 모델

```dart
class ProductMetadata {
  final List<String>? features;
  final List<String>? inclusions;
  final UrgentNotice? urgentNotice;
  final String? promoText;
  final DeliveryInfo? deliveryInfo;

  factory ProductMetadata.fromJson(Map<String, dynamic> json) {
    return ProductMetadata(
      features: (json['features'] as List?)?.cast<String>(),
      inclusions: (json['inclusions'] as List?)?.cast<String>(),
      urgentNotice: json['urgent_notice'] != null
          ? UrgentNotice.fromJson(json['urgent_notice'])
          : null,
      promoText: json['promo_text'],
      deliveryInfo: json['delivery_info'] != null
          ? DeliveryInfo.fromJson(json['delivery_info'])
          : null,
    );
  }
}

class UrgentNotice {
  final String title;
  final List<String> items;

  factory UrgentNotice.fromJson(Map<String, dynamic> json) {
    return UrgentNotice(
      title: json['title'],
      items: (json['items'] as List).cast<String>(),
    );
  }
}

class DeliveryInfo {
  final int estimatedDays;
  final String guarantee;

  factory DeliveryInfo.fromJson(Map<String, dynamic> json) {
    return DeliveryInfo(
      estimatedDays: json['estimated_days'],
      guarantee: json['guarantee'],
    );
  }
}
```

## 5. UI 컴포넌트 구현 가이드

### 5.1 이미지 슬라이더

```dart
class ProductImageSlider extends StatefulWidget {
  final List<String> imageUrls;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // 메인 이미지 (PageView)
        AspectRatio(
          aspectRatio: 1,
          child: PageView.builder(
            itemCount: imageUrls.length,
            onPageChanged: (index) => setState(() => _currentIndex = index),
            itemBuilder: (context, index) => CachedNetworkImage(
              imageUrl: imageUrls[index],
              fit: BoxFit.cover,
            ),
          ),
        ),
        // 인디케이터
        Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: List.generate(
            imageUrls.length,
            (index) => Container(
              width: _currentIndex == index ? 20 : 8,
              height: 8,
              margin: EdgeInsets.symmetric(horizontal: 2),
              decoration: BoxDecoration(
                borderRadius: BorderRadius.circular(4),
                color: _currentIndex == index
                    ? Theme.of(context).primaryColor
                    : Colors.grey[300],
              ),
            ),
          ),
        ),
      ],
    );
  }
}
```

### 5.2 가격 표시 위젯

```dart
class PriceDisplay extends StatelessWidget {
  final double price;
  final double? originalPrice;
  final String unit;

  @override
  Widget build(BuildContext context) {
    final discountPercent = originalPrice != null && originalPrice! > price
        ? ((1 - price / originalPrice!) * 100).round()
        : 0;

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        if (discountPercent > 0) ...[
          Row(
            children: [
              Text(
                '${_formatPrice(originalPrice!)}원',
                style: TextStyle(
                  color: Colors.grey,
                  decoration: TextDecoration.lineThrough,
                  fontSize: 14,
                ),
              ),
              SizedBox(width: 8),
              Container(
                padding: EdgeInsets.symmetric(horizontal: 6, vertical: 2),
                decoration: BoxDecoration(
                  color: Colors.red,
                  borderRadius: BorderRadius.circular(4),
                ),
                child: Text(
                  '$discountPercent% OFF',
                  style: TextStyle(color: Colors.white, fontSize: 12, fontWeight: FontWeight.bold),
                ),
              ),
            ],
          ),
        ],
        Row(
          crossAxisAlignment: CrossAxisAlignment.baseline,
          textBaseline: TextBaseline.alphabetic,
          children: [
            Text(
              _formatPrice(price),
              style: TextStyle(
                fontSize: 28,
                fontWeight: FontWeight.bold,
                color: Theme.of(context).primaryColor,
              ),
            ),
            Text('원/$unit', style: TextStyle(fontSize: 16, color: Colors.grey[600])),
          ],
        ),
      ],
    );
  }

  String _formatPrice(double price) {
    return price.toStringAsFixed(0).replaceAllMapped(
      RegExp(r'(\d{1,3})(?=(\d{3})+(?!\d))'),
      (Match m) => '${m[1]},',
    );
  }
}
```

### 5.3 크기 선택 위젯

```dart
class SizeSelector extends StatelessWidget {
  final List<ProductVariant> sizeVariants;
  final ProductVariant? selectedSize;
  final ValueChanged<ProductVariant> onSelected;

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text('크기 선택', style: TextStyle(fontWeight: FontWeight.w600)),
        SizedBox(height: 8),
        Wrap(
          spacing: 8,
          children: sizeVariants.map((variant) {
            final isSelected = selectedSize?.id == variant.id;
            return GestureDetector(
              onTap: () => onSelected(variant),
              child: Container(
                width: 100,
                padding: EdgeInsets.all(12),
                decoration: BoxDecoration(
                  border: Border.all(
                    color: isSelected ? Theme.of(context).primaryColor : Colors.grey[300]!,
                    width: isSelected ? 2 : 1,
                  ),
                  borderRadius: BorderRadius.circular(12),
                  color: isSelected ? Theme.of(context).primaryColor.withOpacity(0.05) : Colors.white,
                ),
                child: Column(
                  children: [
                    Text(variant.name, style: TextStyle(fontWeight: FontWeight.bold)),
                    if (variant.nameEn != null)
                      Text(variant.nameEn!, style: TextStyle(fontSize: 12, color: Colors.grey)),
                    if (variant.value != null)
                      Text(variant.value!, style: TextStyle(fontSize: 11, color: Colors.grey[400])),
                    if (variant.priceAdjustment != 0)
                      Text(
                        '${variant.priceAdjustment > 0 ? '+' : ''}${_formatPrice(variant.priceAdjustment)}원',
                        style: TextStyle(
                          fontSize: 12,
                          fontWeight: FontWeight.w500,
                          color: variant.priceAdjustment > 0 ? Colors.red : Colors.green,
                        ),
                      ),
                  ],
                ),
              ),
            );
          }).toList(),
        ),
      ],
    );
  }
}
```

### 5.4 수량 선택 위젯

```dart
class QuantitySelector extends StatelessWidget {
  final int quantity;
  final int minQuantity;
  final int maxQuantity;
  final ValueChanged<int> onChanged;

  @override
  Widget build(BuildContext context) {
    return Row(
      children: [
        Text('수량 선택', style: TextStyle(fontWeight: FontWeight.w600)),
        Spacer(),
        Container(
          decoration: BoxDecoration(
            border: Border.all(color: Colors.grey[300]!),
            borderRadius: BorderRadius.circular(8),
          ),
          child: Row(
            children: [
              IconButton(
                icon: Icon(Icons.remove),
                onPressed: quantity > minQuantity
                    ? () => onChanged(quantity - 1)
                    : null,
              ),
              Container(
                width: 40,
                alignment: Alignment.center,
                child: Text('$quantity', style: TextStyle(fontWeight: FontWeight.bold)),
              ),
              IconButton(
                icon: Icon(Icons.add),
                onPressed: quantity < maxQuantity
                    ? () => onChanged(quantity + 1)
                    : null,
              ),
            ],
          ),
        ),
      ],
    );
  }
}
```

### 5.5 탭 컨텐츠

```dart
class ProductDetailTabs extends StatelessWidget {
  final Product product;
  final List<ProductVariant> variants;

  @override
  Widget build(BuildContext context) {
    return DefaultTabController(
      length: 3,
      child: Column(
        children: [
          TabBar(
            tabs: [
              Tab(text: '상품정보'),
              Tab(text: '리뷰'),
              Tab(text: '배송'),
            ],
          ),
          SizedBox(
            height: 400, // 또는 Expanded
            child: TabBarView(
              children: [
                _buildInfoTab(),
                _buildReviewTab(),
                _buildDeliveryTab(),
              ],
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildInfoTab() {
    return SingleChildScrollView(
      padding: EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // 상품 설명
          if (product.description != null) ...[
            Text('상품 설명', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
            SizedBox(height: 8),
            Text(product.description!),
            SizedBox(height: 24),
          ],
          // 특징
          if (product.metadata?.features != null) ...[
            Text('특징', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
            SizedBox(height: 8),
            ...product.metadata!.features!.map((feature) => Padding(
              padding: EdgeInsets.symmetric(vertical: 4),
              child: Row(
                children: [
                  Icon(Icons.check_circle, color: Colors.green, size: 20),
                  SizedBox(width: 8),
                  Expanded(child: Text(feature)),
                ],
              ),
            )),
            SizedBox(height: 24),
          ],
          // 포함 항목
          if (product.metadata?.inclusions != null) ...[
            Text('포함 항목', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
            SizedBox(height: 8),
            ...product.metadata!.inclusions!.asMap().entries.map((entry) => Padding(
              padding: EdgeInsets.symmetric(vertical: 4),
              child: Row(
                children: [
                  CircleAvatar(
                    radius: 12,
                    backgroundColor: Theme.of(context).primaryColor.withOpacity(0.1),
                    child: Text('${entry.key + 1}', style: TextStyle(fontSize: 12)),
                  ),
                  SizedBox(width: 8),
                  Expanded(child: Text(entry.value)),
                ],
              ),
            )),
          ],
        ],
      ),
    );
  }
}
```

## 6. 최종 가격 계산 로직

```dart
class PriceCalculator {
  final Product product;
  final ProductVariant? selectedSize;
  final ProductVariant? selectedCondition;
  final int quantity;

  double get basePrice => product.price;

  double get sizeAdjustment => selectedSize?.priceAdjustment ?? 0;

  double get conditionAdjustment => selectedCondition?.priceAdjustment ?? 0;

  double get unitPrice => basePrice + sizeAdjustment + conditionAdjustment;

  double get totalPrice => unitPrice * quantity;

  // 표시용 문자열
  String get formattedUnitPrice => _formatPrice(unitPrice);
  String get formattedTotalPrice => _formatPrice(totalPrice);

  static String _formatPrice(double price) {
    return price.toStringAsFixed(0).replaceAllMapped(
      RegExp(r'(\d{1,3})(?=(\d{3})+(?!\d))'),
      (Match m) => '${m[1]},',
    );
  }
}
```

## 7. 장바구니 담기 요청

```dart
// 장바구니에 담을 데이터 구조
class CartItem {
  final String productId;
  final String? sizeVariantId;
  final String? conditionVariantId;
  final int quantity;
  final double unitPrice;
  final double totalPrice;

  Map<String, dynamic> toJson() => {
    'product_id': productId,
    'size_variant_id': sizeVariantId,
    'condition_variant_id': conditionVariantId,
    'quantity': quantity,
    'unit_price': unitPrice,
    'total_price': totalPrice,
  };
}

// Supabase에 장바구니 추가
Future<void> addToCart(CartItem item) async {
  await supabase.from('cart_items').insert({
    'user_id': currentUserId,
    ...item.toJson(),
  });
}
```

## 8. 색상 가이드

| 용도 | 색상 | Hex |
|------|------|-----|
| Primary (브랜드) | Blue | #3B82F6 |
| 할인 뱃지 | Red | #EF4444 |
| 추천 뱃지 | Amber | #F59E0B |
| 인기 뱃지 | Rose | #F43F5E |
| 타임세일 뱃지 | Purple | #8B5CF6 |
| 판매중 상태 | Green | #22C55E |
| 품절 상태 | Yellow | #EAB308 |
| 가격 상승 | Red | #DC2626 |
| 가격 할인 | Green | #16A34A |

## 9. 주의사항

1. **이미지 캐싱**: `cached_network_image` 패키지 사용 권장
2. **가격 표시**: 항상 숫자 포맷팅 적용 (천 단위 콤마)
3. **재고 확인**: 수량 선택 시 재고 초과 방지
4. **옵션 필터링**: `is_active: true` 조건 필수
5. **에러 처리**: 네트워크 오류, 품절 상태 등 UI 피드백 필요
6. **로딩 상태**: Skeleton UI 또는 Shimmer 효과 적용 권장

---

*이 문서는 관리자 웹 앱 기준으로 작성되었습니다. 클라이언트 앱 구현 시 장바구니, 주문, 결제 기능을 추가로 구현해야 합니다.*
