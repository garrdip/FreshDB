# 베스트 상품 순위 시스템 - Flutter 클라이언트 가이드

## 개요

이 문서는 Flutter 앱에서 베스트 상품(일간/주간/월간 순위) 화면을 구현하기 위한 가이드입니다.

## UI 구조 (베스트.png 참조)

```
┌─────────────────────────────────────────┐
│         4SEASONS fresh 로고            │
├─────────────────────────────────────────┤
│  [일간순]  [주간순]  [월간순]  ← 기간 탭 │
├─────────────────────────────────────────┤
│  ┌─────────────────────────────────┐   │
│  │ 🥇 1위 상품 (대형 카드)         │   │
│  │ - 이미지, 상품명, 카테고리      │   │
│  │ - 가격, 할인율                  │   │
│  │ - [장바구니] [찜하기] 버튼      │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │ 🥈 2위 상품 (대형 카드)         │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │ 🥉 3위 상품 (대형 카드)         │   │
│  └─────────────────────────────────┘   │
├─────────────────────────────────────────┤
│  ▼ 실시간 랭킹 4위 ~ 7위               │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐  │
│  │ 4위  │ │ 5위  │ │ 6위  │ │ 7위  │  │
│  │ 소형 │ │ 소형 │ │ 소형 │ │ 소형 │  │
│  └──────┘ └──────┘ └──────┘ └──────┘  │
└─────────────────────────────────────────┘
```

---

## 데이터 소스

### 사용 가능한 테이블

| 테이블 | 용도 | 주요 컬럼 |
|--------|------|-----------|
| `products` | 상품 정보 | id, name, price, image_urls, is_popular |
| `orders` | 주문 정보 | id, order_status, created_at |
| `order_items` | 주문 상품 | product_id, quantity, subtotal |
| `user_favorites` | 찜/위시리스트 | product_id, is_wishlisted |
| `categories` | 카테고리 | id, name, product_category_enum |

---

## API 호출 방법

### 방법 1: RPC 함수 사용 (권장)

Supabase에 아래 RPC 함수가 생성되어 있다면:

```dart
// Flutter 코드
Future<List<BestProduct>> fetchBestProducts({
  required String periodType, // 'daily', 'weekly', 'monthly'
  int limit = 10,
}) async {
  final response = await supabase.rpc('get_best_products', params: {
    'period_type': periodType,
    'limit_count': limit,
  });

  return (response as List)
      .map((item) => BestProduct.fromJson(item))
      .toList();
}
```

**Response 구조:**
```json
[
  {
    "rank": 1,
    "product_id": "uuid",
    "product": {
      "id": "uuid",
      "name": "제주 활전복",
      "price": 567000,
      "original_price": 700000,
      "image_urls": ["https://..."],
      "category": "SHELLFISH",
      "freshness_grade": "sashimi",
      "storage_type": "live"
    },
    "sales_count": 150,
    "sales_amount": 85050000,
    "wishlist_count": 45
  }
]
```

---

### 방법 2: 직접 쿼리 (RPC 함수 없을 경우)

```dart
// 일간 베스트 상품 조회
Future<List<Product>> fetchDailyBestProducts() async {
  final now = DateTime.now();
  final oneDayAgo = now.subtract(Duration(days: 1));

  // 1. 기간 내 판매량 집계
  final salesResponse = await supabase
      .from('order_items')
      .select('''
        product_id,
        orders!inner(order_status, created_at)
      ''')
      .gte('orders.created_at', oneDayAgo.toIso8601String())
      .not('orders.order_status', 'in', '(cancelled,refunded)');

  // 2. 판매량 기준 정렬 및 상품 정보 조회
  final productIds = _aggregateSales(salesResponse);

  final productsResponse = await supabase
      .from('products')
      .select('*')
      .inFilter('id', productIds)
      .eq('status', 'active');

  return productsResponse.map((p) => Product.fromJson(p)).toList();
}
```

---

### 방법 3: 단순 인기상품 조회 (is_popular 플래그 사용)

```dart
// 관리자가 지정한 인기상품 조회
Future<List<Product>> fetchPopularProducts({int limit = 10}) async {
  final response = await supabase
      .from('products')
      .select('*')
      .eq('is_popular', true)
      .eq('status', 'active')
      .order('updated_at', ascending: false)
      .limit(limit);

  return (response as List)
      .map((item) => Product.fromJson(item))
      .toList();
}
```

---

## 모델 클래스

### BestProduct Model

```dart
class BestProduct {
  final int rank;
  final String productId;
  final Product product;
  final int salesCount;
  final double salesAmount;
  final int wishlistCount;
  final int? rankChange; // 이전 순위 대비 변동 (선택)

  BestProduct({
    required this.rank,
    required this.productId,
    required this.product,
    required this.salesCount,
    required this.salesAmount,
    required this.wishlistCount,
    this.rankChange,
  });

  factory BestProduct.fromJson(Map<String, dynamic> json) {
    return BestProduct(
      rank: json['rank'] as int,
      productId: json['product_id'] as String,
      product: Product.fromJson(json['product'] as Map<String, dynamic>),
      salesCount: json['sales_count'] as int,
      salesAmount: (json['sales_amount'] as num).toDouble(),
      wishlistCount: json['wishlist_count'] as int,
      rankChange: json['rank_change'] as int?,
    );
  }
}
```

### Product Model (기존)

```dart
class Product {
  final String id;
  final String name;
  final String? description;
  final String category;
  final double price;
  final double? originalPrice;
  final String freshnessGrade;
  final String storageType;
  final String? speciesType;
  final String? origin;
  final String unit;
  final List<String>? imageUrls;
  final bool isFlashSale;
  final bool isPopular;
  final bool isFeatured;
  final String status;

  // 할인율 계산
  int get discountPercent {
    if (originalPrice == null || originalPrice! <= price) return 0;
    return ((1 - price / originalPrice!) * 100).round();
  }

  Product({
    required this.id,
    required this.name,
    this.description,
    required this.category,
    required this.price,
    this.originalPrice,
    required this.freshnessGrade,
    required this.storageType,
    this.speciesType,
    this.origin,
    required this.unit,
    this.imageUrls,
    this.isFlashSale = false,
    this.isPopular = false,
    this.isFeatured = false,
    required this.status,
  });

  factory Product.fromJson(Map<String, dynamic> json) {
    return Product(
      id: json['id'] as String,
      name: json['name'] as String,
      description: json['description'] as String?,
      category: json['category'] as String,
      price: (json['price'] as num).toDouble(),
      originalPrice: json['original_price'] != null
          ? (json['original_price'] as num).toDouble()
          : null,
      freshnessGrade: json['freshness_grade'] as String,
      storageType: json['storage_type'] as String,
      speciesType: json['species_type'] as String?,
      origin: json['origin'] as String?,
      unit: json['unit'] as String,
      imageUrls: (json['image_urls'] as List?)?.cast<String>(),
      isFlashSale: json['is_flash_sale'] as bool? ?? false,
      isPopular: json['is_popular'] as bool? ?? false,
      isFeatured: json['is_featured'] as bool? ?? false,
      status: json['status'] as String,
    );
  }
}
```

---

## UI 구현 가이드

### 기간 탭 (SegmentedButton)

```dart
enum RankingPeriod { daily, weekly, monthly }

class BestProductsPage extends StatefulWidget {
  @override
  State<BestProductsPage> createState() => _BestProductsPageState();
}

class _BestProductsPageState extends State<BestProductsPage> {
  RankingPeriod _selectedPeriod = RankingPeriod.daily;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // 기간 탭
        SegmentedButton<RankingPeriod>(
          segments: const [
            ButtonSegment(value: RankingPeriod.daily, label: Text('일간순')),
            ButtonSegment(value: RankingPeriod.weekly, label: Text('주간순')),
            ButtonSegment(value: RankingPeriod.monthly, label: Text('월간순')),
          ],
          selected: {_selectedPeriod},
          onSelectionChanged: (Set<RankingPeriod> selection) {
            setState(() {
              _selectedPeriod = selection.first;
            });
          },
        ),

        // 상품 목록
        Expanded(
          child: _buildProductList(),
        ),
      ],
    );
  }
}
```

### TOP 3 대형 카드

```dart
Widget _buildTopRankCard(BestProduct item) {
  final product = item.product;

  return Card(
    child: Column(
      children: [
        // 순위 배지
        Positioned(
          top: 8,
          left: 8,
          child: Container(
            padding: EdgeInsets.symmetric(horizontal: 12, vertical: 6),
            decoration: BoxDecoration(
              color: _getRankColor(item.rank),
              borderRadius: BorderRadius.circular(4),
            ),
            child: Text(
              '${item.rank}',
              style: TextStyle(
                color: Colors.white,
                fontWeight: FontWeight.bold,
              ),
            ),
          ),
        ),

        // 상품 이미지
        AspectRatio(
          aspectRatio: 16 / 9,
          child: Image.network(
            product.imageUrls?.first ?? '',
            fit: BoxFit.cover,
          ),
        ),

        // 상품 정보
        Padding(
          padding: EdgeInsets.all(12),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              // 카테고리
              Text(
                _getCategoryName(product.category),
                style: TextStyle(color: Colors.grey),
              ),

              // 상품명
              Text(
                product.name,
                style: TextStyle(
                  fontSize: 16,
                  fontWeight: FontWeight.bold,
                ),
              ),

              // 가격
              Row(
                children: [
                  Text(
                    '${NumberFormat('#,###').format(product.price)}원',
                    style: TextStyle(
                      fontSize: 18,
                      fontWeight: FontWeight.bold,
                      color: Theme.of(context).primaryColor,
                    ),
                  ),
                  if (product.discountPercent > 0) ...[
                    SizedBox(width: 8),
                    Text(
                      '${product.discountPercent}%',
                      style: TextStyle(
                        color: Colors.red,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                  ],
                ],
              ),

              // 버튼
              Row(
                children: [
                  Expanded(
                    child: OutlinedButton(
                      onPressed: () => _addToCart(product),
                      child: Text('장바구니'),
                    ),
                  ),
                  SizedBox(width: 8),
                  OutlinedButton(
                    onPressed: () => _toggleWishlist(product),
                    child: Text('찜하기'),
                  ),
                ],
              ),
            ],
          ),
        ),
      ],
    ),
  );
}

Color _getRankColor(int rank) {
  switch (rank) {
    case 1: return Color(0xFFFFD700); // Gold
    case 2: return Color(0xFFC0C0C0); // Silver
    case 3: return Color(0xFFCD7F32); // Bronze
    default: return Colors.grey;
  }
}
```

### 4위 이하 그리드

```dart
Widget _buildLowerRankGrid(List<BestProduct> items) {
  // 4위부터 표시
  final lowerItems = items.where((i) => i.rank > 3).toList();

  return Column(
    crossAxisAlignment: CrossAxisAlignment.start,
    children: [
      Padding(
        padding: EdgeInsets.all(16),
        child: Text(
          '▼ 실시간 랭킹 4위 ~ ${lowerItems.length + 3}위',
          style: TextStyle(
            fontSize: 14,
            fontWeight: FontWeight.w500,
          ),
        ),
      ),
      GridView.builder(
        shrinkWrap: true,
        physics: NeverScrollableScrollPhysics(),
        gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
          crossAxisCount: 2,
          childAspectRatio: 0.75,
          crossAxisSpacing: 8,
          mainAxisSpacing: 8,
        ),
        itemCount: lowerItems.length,
        itemBuilder: (context, index) {
          return _buildSmallRankCard(lowerItems[index]);
        },
      ),
    ],
  );
}

Widget _buildSmallRankCard(BestProduct item) {
  return Card(
    child: Column(
      children: [
        // 순위 + 이미지
        Stack(
          children: [
            AspectRatio(
              aspectRatio: 1,
              child: Image.network(
                item.product.imageUrls?.first ?? '',
                fit: BoxFit.cover,
              ),
            ),
            Positioned(
              top: 4,
              left: 4,
              child: Container(
                padding: EdgeInsets.all(4),
                color: Colors.black54,
                child: Text(
                  '${item.rank}위',
                  style: TextStyle(
                    color: Colors.white,
                    fontSize: 12,
                  ),
                ),
              ),
            ),
          ],
        ),

        // 상품명 + 가격
        Padding(
          padding: EdgeInsets.all(8),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(
                item.product.name,
                maxLines: 1,
                overflow: TextOverflow.ellipsis,
              ),
              Text(
                '${NumberFormat('#,###').format(item.product.price)}원',
                style: TextStyle(fontWeight: FontWeight.bold),
              ),
            ],
          ),
        ),
      ],
    ),
  );
}
```

---

## 카테고리 매핑

```dart
String getCategoryName(String categoryEnum) {
  const categoryNames = {
    'FISH': '어류',
    'SHELLFISH': '조개류',
    'CRUSTACEAN': '갑각류',
    'MOLLUSK': '연체류',
  };
  return categoryNames[categoryEnum] ?? categoryEnum;
}
```

---

## 캐싱 전략

```dart
class BestProductsProvider extends ChangeNotifier {
  Map<RankingPeriod, List<BestProduct>> _cache = {};
  Map<RankingPeriod, DateTime> _cacheTimestamp = {};

  static const _cacheDuration = Duration(minutes: 5);

  Future<List<BestProduct>> getBestProducts(RankingPeriod period) async {
    // 캐시 확인
    if (_cache.containsKey(period)) {
      final timestamp = _cacheTimestamp[period];
      if (timestamp != null &&
          DateTime.now().difference(timestamp) < _cacheDuration) {
        return _cache[period]!;
      }
    }

    // API 호출
    final products = await _fetchFromApi(period);

    // 캐시 저장
    _cache[period] = products;
    _cacheTimestamp[period] = DateTime.now();

    notifyListeners();
    return products;
  }

  void invalidateCache() {
    _cache.clear();
    _cacheTimestamp.clear();
  }
}
```

---

## 필요한 DB 마이그레이션

### 추가 테이블: 없음

현재 DB 구조로 충분히 구현 가능합니다.

### RPC 함수 추가 (선택적, 성능 최적화용)

관리자에게 아래 SQL 실행 요청:

```sql
-- 베스트 상품 조회 함수
CREATE OR REPLACE FUNCTION get_best_products(
  period_type TEXT DEFAULT 'daily',
  limit_count INT DEFAULT 10
)
RETURNS TABLE (
  rank BIGINT,
  product_id UUID,
  product JSONB,
  sales_count BIGINT,
  sales_amount NUMERIC,
  wishlist_count BIGINT
)
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
  RETURN QUERY
  WITH sales_stats AS (
    SELECT
      oi.product_id,
      COUNT(*) as sales_cnt,
      COALESCE(SUM(oi.subtotal), 0) as sales_amt
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.id
    WHERE o.order_status NOT IN ('cancelled', 'refunded')
      AND o.created_at >= CASE
        WHEN period_type = 'daily' THEN NOW() - INTERVAL '1 day'
        WHEN period_type = 'weekly' THEN NOW() - INTERVAL '7 days'
        WHEN period_type = 'monthly' THEN NOW() - INTERVAL '30 days'
        ELSE NOW() - INTERVAL '1 day'
      END
    GROUP BY oi.product_id
  ),
  wishlist_stats AS (
    SELECT
      product_id,
      COUNT(*) as wish_cnt
    FROM user_favorites
    WHERE is_wishlisted = true
    GROUP BY product_id
  )
  SELECT
    ROW_NUMBER() OVER (
      ORDER BY COALESCE(s.sales_cnt, 0) DESC,
               COALESCE(w.wish_cnt, 0) DESC
    )::BIGINT as rank,
    p.id as product_id,
    to_jsonb(p) as product,
    COALESCE(s.sales_cnt, 0)::BIGINT as sales_count,
    COALESCE(s.sales_amt, 0) as sales_amount,
    COALESCE(w.wish_cnt, 0)::BIGINT as wishlist_count
  FROM products p
  LEFT JOIN sales_stats s ON p.id = s.product_id
  LEFT JOIN wishlist_stats w ON p.id = w.product_id
  WHERE p.status = 'active'
  ORDER BY COALESCE(s.sales_cnt, 0) DESC, COALESCE(w.wish_cnt, 0) DESC
  LIMIT limit_count;
END;
$$;
```

---

## 요약

| 항목 | 상태 |
|------|------|
| 추가 테이블 | 불필요 |
| RPC 함수 | 선택적 (성능 최적화용) |
| Edge Function | 불필요 |
| 실시간 업데이트 | 클라이언트 캐싱으로 대체 |

**권장 구현 순서:**
1. 먼저 `products.is_popular` 기반 단순 구현
2. 주문 데이터 충분히 쌓이면 RPC 함수로 전환
3. 필요시 관리자 페이지에서 수동 순위 조정 기능 추가
