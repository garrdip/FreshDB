# Flutter 홈 화면 API 가이드

FreshMall Flutter 앱의 홈 화면 구현을 위한 Supabase 데이터베이스 API 문서입니다.

---

## Supabase 연결 정보

```dart
final supabase = Supabase.instance.client;

// 프로젝트 URL
const supabaseUrl = 'https://niifwjpbdutjdsswjrtv.supabase.co';
```

---

## 홈 화면 UI 구성요소

```
┌─────────────────────────────────────┐
│  [검색창]                           │
│                                     │
│  인기 검색어: 문어 전복 크랩 우럭   │  ← popular_searches
│                                     │
│  ┌─────────────────────────────┐   │
│  │ 카테고리 바로가기            │   │  ← categories
│  │ 전체 어류 조개 갑각 연체 ... │   │
│  └─────────────────────────────┘   │
│                                     │
│  재발주 & 단골상품                  │  ← user_favorites + products
│  [상품카드] [상품카드] [상품카드]   │
│                                     │
│  놓치기 아까운 특가                 │  ← products (is_flash_sale)
│  [상품카드] [상품카드] [상품카드]   │
└─────────────────────────────────────┘
```

---

## 1. 카테고리 (categories)

### 테이블 스키마

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | uuid | PK |
| `name` | text | 카테고리명 (한글) |
| `name_en` | text? | 영문명 |
| `slug` | text | URL용 슬러그 |
| `icon_url` | text? | 아이콘 이미지 URL |
| `category_type` | text | `main` / `special` |
| `product_category_enum` | text? | products.category 매핑값 |
| `filter_condition` | jsonb? | 특수 카테고리 필터 조건 |
| `display_order` | int | 표시 순서 |
| `is_active` | bool | 활성 여부 |
| `show_in_home` | bool | 홈 화면 표시 여부 |

### Flutter 쿼리

```dart
/// 홈 화면 카테고리 목록 조회
Future<List<Category>> getHomeCategories() async {
  final response = await supabase
      .from('categories')
      .select()
      .eq('show_in_home', true)
      .eq('is_active', true)
      .order('display_order', ascending: true);

  return (response as List).map((e) => Category.fromJson(e)).toList();
}
```

### Dart 모델

```dart
class Category {
  final String id;
  final String name;
  final String? nameEn;
  final String slug;
  final String? iconUrl;
  final String categoryType; // 'main' | 'special'
  final String? productCategoryEnum;
  final Map<String, dynamic>? filterCondition;
  final int displayOrder;
  final bool isActive;
  final bool showInHome;

  Category({
    required this.id,
    required this.name,
    this.nameEn,
    required this.slug,
    this.iconUrl,
    required this.categoryType,
    this.productCategoryEnum,
    this.filterCondition,
    required this.displayOrder,
    required this.isActive,
    required this.showInHome,
  });

  factory Category.fromJson(Map<String, dynamic> json) => Category(
    id: json['id'],
    name: json['name'],
    nameEn: json['name_en'],
    slug: json['slug'],
    iconUrl: json['icon_url'],
    categoryType: json['category_type'],
    productCategoryEnum: json['product_category_enum'],
    filterCondition: json['filter_condition'],
    displayOrder: json['display_order'] ?? 0,
    isActive: json['is_active'] ?? true,
    showInHome: json['show_in_home'] ?? true,
  );
}
```

### 카테고리별 상품 조회

```dart
/// 카테고리 슬러그로 상품 목록 조회
Future<List<Product>> getProductsByCategory(Category category) async {
  var query = supabase
      .from('products')
      .select()
      .eq('status', 'active');

  // 메인 카테고리: product_category_enum으로 필터
  if (category.categoryType == 'main' && category.productCategoryEnum != null) {
    query = query.eq('category', category.productCategoryEnum!);
  }
  // 특수 카테고리: filter_condition 적용
  else if (category.categoryType == 'special' && category.filterCondition != null) {
    // 예: {"is_popular": true}
    category.filterCondition!.forEach((key, value) {
      query = query.eq(key, value);
    });
  }
  // 신상품: 최근 7일
  else if (category.slug == 'new') {
    final weekAgo = DateTime.now().subtract(Duration(days: 7)).toIso8601String();
    query = query.gte('created_at', weekAgo);
  }

  final response = await query.order('created_at', ascending: false);
  return (response as List).map((e) => Product.fromJson(e)).toList();
}
```

---

## 2. 인기 검색어 (popular_searches)

### 테이블 스키마

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | uuid | PK |
| `keyword` | text | 검색 키워드 |
| `search_count` | bigint | 총 검색 횟수 |
| `daily_count` | bigint | 일일 검색 횟수 |
| `weekly_count` | bigint | 주간 검색 횟수 |
| `ranking` | int? | 현재 순위 |
| `previous_ranking` | int? | 이전 순위 (변동 표시용) |
| `linked_category_id` | uuid? | 연결된 카테고리 ID |
| `linked_product_id` | uuid? | 연결된 상품 ID |
| `is_active` | bool | 활성 여부 |
| `is_featured` | bool | 추천 검색어 여부 |

### Flutter 쿼리

```dart
/// 인기 검색어 조회 (상위 N개)
Future<List<PopularSearch>> getPopularSearches({int limit = 4}) async {
  final response = await supabase
      .from('popular_searches')
      .select()
      .eq('is_active', true)
      .not('ranking', 'is', null)
      .order('ranking', ascending: true)
      .limit(limit);

  return (response as List).map((e) => PopularSearch.fromJson(e)).toList();
}

/// 검색 실행 시 카운트 증가 (RPC 호출)
Future<void> incrementSearchCount(String keyword) async {
  await supabase.rpc('increment_search_count', params: {'p_keyword': keyword});
}
```

### Dart 모델

```dart
class PopularSearch {
  final String id;
  final String keyword;
  final int searchCount;
  final int dailyCount;
  final int weeklyCount;
  final int? ranking;
  final int? previousRanking;
  final String? linkedCategoryId;
  final String? linkedProductId;
  final bool isActive;
  final bool isFeatured;

  PopularSearch({
    required this.id,
    required this.keyword,
    required this.searchCount,
    required this.dailyCount,
    required this.weeklyCount,
    this.ranking,
    this.previousRanking,
    this.linkedCategoryId,
    this.linkedProductId,
    required this.isActive,
    required this.isFeatured,
  });

  /// 순위 변동 계산 (상승: 양수, 하락: 음수)
  int? get rankingChange {
    if (ranking == null || previousRanking == null) return null;
    return previousRanking! - ranking!;
  }

  factory PopularSearch.fromJson(Map<String, dynamic> json) => PopularSearch(
    id: json['id'],
    keyword: json['keyword'],
    searchCount: json['search_count'] ?? 0,
    dailyCount: json['daily_count'] ?? 0,
    weeklyCount: json['weekly_count'] ?? 0,
    ranking: json['ranking'],
    previousRanking: json['previous_ranking'],
    linkedCategoryId: json['linked_category_id'],
    linkedProductId: json['linked_product_id'],
    isActive: json['is_active'] ?? true,
    isFeatured: json['is_featured'] ?? false,
  );
}
```

### 데이터베이스 함수

```sql
-- 검색 횟수 증가 (자동 upsert)
increment_search_count(p_keyword TEXT) → void
```

---

## 3. 단골상품/찜 (user_favorites)

### 테이블 스키마

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | uuid | PK |
| `user_id` | uuid | 사용자 ID (FK → users) |
| `product_id` | uuid | 상품 ID (FK → products) |
| `order_count` | int | 주문 횟수 |
| `total_quantity` | numeric | 총 주문 수량 |
| `total_amount` | numeric | 총 주문 금액 |
| `last_ordered_at` | timestamptz? | 마지막 주문일 |
| `is_wishlisted` | bool | 찜 여부 |
| `wishlisted_at` | timestamptz? | 찜한 날짜 |

### Flutter 쿼리

```dart
/// 재발주 & 단골상품 조회 (2회 이상 주문)
Future<List<FavoriteProduct>> getFrequentProducts({int limit = 4}) async {
  final userId = supabase.auth.currentUser?.id;
  if (userId == null) return [];

  final response = await supabase
      .from('user_favorites')
      .select('''
        *,
        products (*)
      ''')
      .eq('user_id', userId)
      .gte('order_count', 2)
      .order('order_count', ascending: false)
      .order('last_ordered_at', ascending: false)
      .limit(limit);

  return (response as List).map((e) => FavoriteProduct.fromJson(e)).toList();
}

/// 찜 목록 조회
Future<List<FavoriteProduct>> getWishlist() async {
  final userId = supabase.auth.currentUser?.id;
  if (userId == null) return [];

  final response = await supabase
      .from('user_favorites')
      .select('''
        *,
        products (*)
      ''')
      .eq('user_id', userId)
      .eq('is_wishlisted', true)
      .order('wishlisted_at', ascending: false);

  return (response as List).map((e) => FavoriteProduct.fromJson(e)).toList();
}

/// 찜하기 토글
Future<void> toggleWishlist(String productId) async {
  final userId = supabase.auth.currentUser?.id;
  if (userId == null) throw Exception('로그인이 필요합니다');

  // 기존 레코드 확인
  final existing = await supabase
      .from('user_favorites')
      .select()
      .eq('user_id', userId)
      .eq('product_id', productId)
      .maybeSingle();

  if (existing == null) {
    // 새로 추가
    await supabase.from('user_favorites').insert({
      'user_id': userId,
      'product_id': productId,
      'is_wishlisted': true,
      'wishlisted_at': DateTime.now().toIso8601String(),
    });
  } else {
    // 토글
    await supabase
        .from('user_favorites')
        .update({
          'is_wishlisted': !(existing['is_wishlisted'] ?? false),
          'wishlisted_at': DateTime.now().toIso8601String(),
        })
        .eq('id', existing['id']);
  }
}
```

### Dart 모델

```dart
class FavoriteProduct {
  final String id;
  final String userId;
  final String productId;
  final int orderCount;
  final double totalQuantity;
  final double totalAmount;
  final DateTime? lastOrderedAt;
  final bool isWishlisted;
  final DateTime? wishlistedAt;
  final Product? product; // JOIN된 상품 정보

  FavoriteProduct({
    required this.id,
    required this.userId,
    required this.productId,
    required this.orderCount,
    required this.totalQuantity,
    required this.totalAmount,
    this.lastOrderedAt,
    required this.isWishlisted,
    this.wishlistedAt,
    this.product,
  });

  factory FavoriteProduct.fromJson(Map<String, dynamic> json) => FavoriteProduct(
    id: json['id'],
    userId: json['user_id'],
    productId: json['product_id'],
    orderCount: json['order_count'] ?? 0,
    totalQuantity: (json['total_quantity'] ?? 0).toDouble(),
    totalAmount: (json['total_amount'] ?? 0).toDouble(),
    lastOrderedAt: json['last_ordered_at'] != null
        ? DateTime.parse(json['last_ordered_at'])
        : null,
    isWishlisted: json['is_wishlisted'] ?? false,
    wishlistedAt: json['wishlisted_at'] != null
        ? DateTime.parse(json['wishlisted_at'])
        : null,
    product: json['products'] != null
        ? Product.fromJson(json['products'])
        : null,
  );
}
```

### 자동 업데이트

> **참고**: `user_favorites`는 주문이 `confirmed` 상태로 변경될 때 트리거에 의해 자동 업데이트됩니다. 클라이언트에서 별도 처리 불필요.

---

## 4. 상품 (products)

### 테이블 스키마 (홈 화면 관련 필드)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | uuid | PK |
| `sku` | varchar | SKU 코드 |
| `name` | varchar | 상품명 |
| `description` | text? | 상품 설명 |
| `category` | varchar | 카테고리 (FISH, SHELLFISH 등) |
| `price` | numeric | 판매가 |
| `original_price` | numeric? | 원래가 (할인 표시용) |
| `freshness_grade` | varchar | 선도 등급 |
| `storage_type` | varchar | 세부 종류/보관상태 |
| `origin` | varchar? | 원산지 |
| `company` | varchar? | 공급회사 |
| `unit` | varchar | 판매 단위 (kg, g, piece, box) |
| `min_quantity` | numeric | 최소 주문 수량 |
| `stock_quantity` | numeric | 재고 수량 |
| `image_urls` | text[] | 이미지 URL 배열 |
| `is_flash_sale` | bool | 타임세일 여부 |
| `flash_sale_start` | timestamptz? | 타임세일 시작 |
| `flash_sale_end` | timestamptz? | 타임세일 종료 |
| `is_popular` | bool | 인기 상품 여부 |
| `is_featured` | bool | 추천 상품 여부 |
| `status` | varchar | 상태 (active, inactive, out_of_stock, deleted) |

### category ENUM 값

| 값 | 설명 |
|----|------|
| `FISH` | 어류 |
| `SHELLFISH` | 조개류 |
| `CRUSTACEAN` | 갑각류 |
| `MOLLUSK` | 연체류 |
| `FLASH_SALE` | 타임세일 |
| `POPULAR` | 인기상품 |

### storage_type ENUM 값

| 카테고리 | 값 | 설명 |
|----------|-----|------|
| MOLLUSK | squid | 오징어 |
| MOLLUSK | octopus | 문어 |
| MOLLUSK | nakji | 낙지 |
| CRUSTACEAN | crab | 대게 |
| CRUSTACEAN | shrimp | 새우 |
| CRUSTACEAN | lobster | 가재 |
| SHELLFISH | abalone | 전복 |
| SHELLFISH | pen_shell | 키조개 |
| SHELLFISH | surf_clam | 모시조개 |
| SHELLFISH | mussel | 홍합 |
| 기본 | live | 활어 |
| 기본 | fresh | 선어 |
| 기본 | frozen | 냉동 |

### Flutter 쿼리

```dart
/// 타임세일 상품 조회
Future<List<Product>> getFlashSaleProducts({int limit = 4}) async {
  final now = DateTime.now().toIso8601String();

  final response = await supabase
      .from('products')
      .select()
      .eq('is_flash_sale', true)
      .eq('status', 'active')
      .or('flash_sale_start.is.null,flash_sale_start.lte.$now')
      .or('flash_sale_end.is.null,flash_sale_end.gte.$now')
      .order('updated_at', ascending: false)
      .limit(limit);

  return (response as List).map((e) => Product.fromJson(e)).toList();
}

/// 인기 상품 조회
Future<List<Product>> getPopularProducts({int limit = 4}) async {
  final response = await supabase
      .from('products')
      .select()
      .eq('is_popular', true)
      .eq('status', 'active')
      .order('updated_at', ascending: false)
      .limit(limit);

  return (response as List).map((e) => Product.fromJson(e)).toList();
}

/// 신상품 조회 (최근 7일)
Future<List<Product>> getNewProducts({int limit = 4}) async {
  final weekAgo = DateTime.now().subtract(Duration(days: 7)).toIso8601String();

  final response = await supabase
      .from('products')
      .select()
      .eq('status', 'active')
      .gte('created_at', weekAgo)
      .order('created_at', ascending: false)
      .limit(limit);

  return (response as List).map((e) => Product.fromJson(e)).toList();
}

/// 상품 검색
Future<List<Product>> searchProducts(String query) async {
  final response = await supabase
      .from('products')
      .select()
      .eq('status', 'active')
      .or('name.ilike.%$query%,description.ilike.%$query%,sku.ilike.%$query%')
      .order('is_popular', ascending: false)
      .limit(20);

  return (response as List).map((e) => Product.fromJson(e)).toList();
}
```

### Dart 모델

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
  final double minQuantity;
  final double stockQuantity;
  final List<String> imageUrls;
  final bool isFlashSale;
  final DateTime? flashSaleStart;
  final DateTime? flashSaleEnd;
  final bool isPopular;
  final bool isFeatured;
  final String status;
  final DateTime createdAt;
  final DateTime updatedAt;

  Product({
    required this.id,
    required this.sku,
    required this.name,
    this.description,
    required this.category,
    required this.price,
    this.originalPrice,
    required this.freshnessGrade,
    required this.storageType,
    this.origin,
    this.company,
    required this.unit,
    required this.minQuantity,
    required this.stockQuantity,
    required this.imageUrls,
    required this.isFlashSale,
    this.flashSaleStart,
    this.flashSaleEnd,
    required this.isPopular,
    required this.isFeatured,
    required this.status,
    required this.createdAt,
    required this.updatedAt,
  });

  /// 할인율 계산
  int get discountRate {
    if (originalPrice == null || originalPrice! <= price) return 0;
    return ((originalPrice! - price) / originalPrice! * 100).round();
  }

  /// 타임세일 진행 중 여부
  bool get isFlashSaleActive {
    if (!isFlashSale) return false;
    final now = DateTime.now();
    if (flashSaleStart != null && now.isBefore(flashSaleStart!)) return false;
    if (flashSaleEnd != null && now.isAfter(flashSaleEnd!)) return false;
    return true;
  }

  /// 신상품 여부 (7일 이내)
  bool get isNew {
    return DateTime.now().difference(createdAt).inDays <= 7;
  }

  /// 대표 이미지
  String? get thumbnailUrl => imageUrls.isNotEmpty ? imageUrls.first : null;

  factory Product.fromJson(Map<String, dynamic> json) => Product(
    id: json['id'],
    sku: json['sku'],
    name: json['name'],
    description: json['description'],
    category: json['category'],
    price: (json['price'] ?? 0).toDouble(),
    originalPrice: json['original_price']?.toDouble(),
    freshnessGrade: json['freshness_grade'],
    storageType: json['storage_type'],
    origin: json['origin'],
    company: json['company'],
    unit: json['unit'] ?? 'kg',
    minQuantity: (json['min_quantity'] ?? 1).toDouble(),
    stockQuantity: (json['stock_quantity'] ?? 0).toDouble(),
    imageUrls: List<String>.from(json['image_urls'] ?? []),
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
    createdAt: DateTime.parse(json['created_at']),
    updatedAt: DateTime.parse(json['updated_at']),
  );
}
```

---

## 5. 상품 뱃지 표시 로직

```dart
enum ProductBadge {
  popular,   // 인기
  featured,  // 도매
  flashSale, // 특가
  newProduct,// 신상품
}

extension ProductBadges on Product {
  List<ProductBadge> get badges {
    final result = <ProductBadge>[];
    if (isPopular) result.add(ProductBadge.popular);
    if (isFeatured) result.add(ProductBadge.featured);
    if (isFlashSaleActive) result.add(ProductBadge.flashSale);
    if (isNew) result.add(ProductBadge.newProduct);
    return result;
  }
}

// 뱃지 스타일
Map<ProductBadge, Map<String, dynamic>> badgeStyles = {
  ProductBadge.popular: {'text': '인기', 'color': Colors.blue},
  ProductBadge.featured: {'text': '도매', 'color': Colors.blue},
  ProductBadge.flashSale: {'text': '특가', 'color': Colors.red},
  ProductBadge.newProduct: {'text': '신상품', 'color': Colors.green},
};
```

---

## 6. 홈 화면 통합 데이터 로드

```dart
class HomeScreenData {
  final List<Category> categories;
  final List<PopularSearch> popularSearches;
  final List<FavoriteProduct> frequentProducts;
  final List<Product> flashSaleProducts;

  HomeScreenData({
    required this.categories,
    required this.popularSearches,
    required this.frequentProducts,
    required this.flashSaleProducts,
  });
}

/// 홈 화면 데이터 병렬 로드
Future<HomeScreenData> loadHomeScreenData() async {
  final results = await Future.wait([
    getHomeCategories(),
    getPopularSearches(limit: 4),
    getFrequentProducts(limit: 4),
    getFlashSaleProducts(limit: 4),
  ]);

  return HomeScreenData(
    categories: results[0] as List<Category>,
    popularSearches: results[1] as List<PopularSearch>,
    frequentProducts: results[2] as List<FavoriteProduct>,
    flashSaleProducts: results[3] as List<Product>,
  );
}
```

---

## 7. RLS (Row Level Security) 정책

### 공개 읽기 테이블
- `categories`: `is_active = true` 조건으로 모든 사용자 조회 가능
- `popular_searches`: `is_active = true` 조건으로 모든 사용자 조회 가능
- `products`: `status != 'deleted'` 조건으로 모든 사용자 조회 가능

### 사용자 전용 테이블
- `user_favorites`: 본인 데이터만 CRUD 가능 (`user_id = auth.uid()`)

---

## 8. 데이터베이스 함수 목록

| 함수명 | 반환 타입 | 용도 |
|--------|----------|------|
| `increment_search_count(p_keyword)` | void | 검색어 카운트 증가 |
| `update_popular_search_rankings()` | void | 인기 검색어 순위 재계산 (관리자/cron) |
| `is_admin()` | boolean | 현재 사용자 관리자 여부 확인 |

---

## 9. 참고 사항

### Storage 버킷
- `product-images`: 상품 이미지
- `category-icons`: 카테고리 아이콘

### 이미지 URL 형식
```
https://niifwjpbdutjdsswjrtv.supabase.co/storage/v1/object/public/product-images/{product_id}/{filename}
https://niifwjpbdutjdsswjrtv.supabase.co/storage/v1/object/public/category-icons/{filename}
```

### 에러 처리
```dart
try {
  final data = await loadHomeScreenData();
  // 성공 처리
} on PostgrestException catch (e) {
  // Supabase 에러
  print('Database error: ${e.message}');
} catch (e) {
  // 기타 에러
  print('Error: $e');
}
```

---

## 변경 이력

| 날짜 | 변경 내용 |
|------|----------|
| 2025-01-07 | 초기 문서 작성 |
