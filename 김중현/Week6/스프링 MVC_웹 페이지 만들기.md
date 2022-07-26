# Section 7. 스프링 MVC - 웹 페이지 만들기
> 상품 관리 서비스를 제작해보자!

## 요구사항 분석
> 상품 관리 서비스 
#### 상품 도메인 모델
```
상품 ID
상품명
가격
수량
```
<br>
<br>

#### 상품 관리 기능
```
상품 목록
상품 상세
상품 등록
상품 수정
```
<br>
<br>

#### 서비스 제공 흐름 정리
<img width="700" alt="스크린샷 2022-09-01 오후 12 34 21" src="https://user-images.githubusercontent.com/80838501/187830014-17c36994-b703-4ebd-ae0b-2e1b45894542.png">
<br>
<br>
<br>
<br>

## 상품 도메인 개발
#### Item
> 상품 객체
```java
package hello.itemservice.domain.item;

import lombok.Data;

@Data
public class Item {
    private Long id;
    private String itemName;
    private Integer price;
    private Integer quantity;

    public Item() {
    }

    public Item(String itemName, Integer price, Integer quantity) {
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    }
}
```
- 상품 ID, 상품명, 가격, 수량
- 생성자
<br>
<br>

#### ItemRepository
> 상품 저장소
```java
package hello.itemservice.domain.item;

import org.springframework.stereotype.Repository;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Repository
public class ItemRepository {

    private static final Map<Long, Item> store = new HashMap<>();
    private static long sequence = 0L;

    public Item save(Item item) {
        item.setId(++sequence);
        store.put(item.getId(), item);

        return item;
    }

    public Item findById(Long id) {
        return store.get(id);
    }

    public List<Item> findAll() {
        return new ArrayList<>(store.values());
    }

    public void update(Long itemId, Item updateParam) {
        Item findItem = findById(itemId);
        findItem.setItemName(updateParam.getItemName());
        findItem.setPrice(updateParam.getPrice());
        findItem.setQuantity(updateParam.getQuantity());
    }

    public void clearStore() {
        store.clear();
    }
}
```
- 저장, Id로 조회, 전체 조회, 수정, 전체 clear(테스트용)
<br>
<br>

#### ItemRepositoryTest
> 상품 저장소 테스트
```java
package hello.itemservice.domain.item;

import org.assertj.core.api.Assertions;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.*;

public class ItemRepositoryTest {

    ItemRepository itemRepository = new ItemRepository();

    @AfterEach
    void afterEach() { //테스트 돌 때마다 clear
        itemRepository.clearStore();
    }

    @Test
    void save() {
        //given
        Item item = new Item("itemA", 10000, 10);

        //when
        Item savedItem = itemRepository.save(item);

        //then
        Item findItem = itemRepository.findById(item.getId());
        assertThat(findItem).isEqualTo(savedItem);
    }

    @Test
    void findAll() {
        //given
        Item item1 = new Item("item1", 10000, 10);
        Item item2 = new Item("item2", 20000, 20);

        itemRepository.save(item1);
        itemRepository.save(item2);

        //when
        List<Item> result = itemRepository.findAll();

        //then
        assertThat(result.size()).isEqualTo(2);
        assertThat(result).contains(item1, item2);
    }

    @Test
    void updateItem() {
        //given
        Item item = new Item("item1", 10000, 10);

        Item savedItem = itemRepository.save(item);
        Long itemId = savedItem.getId();

        //when
        Item updateParam = new Item("item2", 20000, 30);
        itemRepository.update(itemId, updateParam);

        //then
        Item findItem = itemRepository.findById(itemId);

        assertThat(findItem.getItemName()).isEqualTo(updateParam.getItemName());
        assertThat(findItem.getPrice()).isEqualTo(updateParam.getPrice());
        assertThat(findItem.getQuantity()).isEqualTo(updateParam.getQuantity());
    }
}
```
- 저장, 전체 조회, 수정 테스트 
<br>
<br>
<br>
<br>

## 상품 목록 - 타임리프
#### BasicItemController
```java
package hello.itemservice.web.basic;

import hello.itemservice.domain.item.Item;
import hello.itemservice.domain.item.ItemRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;

import javax.annotation.PostConstruct;
import java.util.List;

@Controller
@RequestMapping("/basic/items")
@RequiredArgsConstructor
public class BasicItemController {

    private final ItemRepository itemRepository;

    @GetMapping
    public String items(Model model) { // "/basic/items" 경로로 GET이 오면 이 메소드가 호출
        List<Item> items = itemRepository.findAll();
        model.addAttribute("items", items); //model에 "items"라고 컬렉션이 담긴다.
        return "basic/item"; //view 보여주기
    }

    /**
     * 테스트용 데이터 추가
     */
    @PostConstruct
    public void init() {
        itemRepository.save(new Item("itemA", 10000, 10));
        itemRepository.save(new Item("itemB", 20000, 20));
    }
}
```
- itemRepository에서 모든 상품을 조회한 뒤 모델에 담고, 뷰 템플릿을 호출한다.
- @PostConstruct: 헤당 빈의 의존관계가 모두 주입되고 나면 초기화를 위해 호출된다.
- `@RequiredArgsConstructor` 애노테이션: final이 붙은 필드의 생성자를 자동 생성해주는 롬복 애노테이션
```java
@Controller
@RequestMapping("/basic/items")
public class BasicItemController {

    private final ItemRepository itemRepository;

    @Autowired
    public BasicItemController(ItemRepository itemRepository) {
        this.itemRepository = itemRepository;
    }
}
```

```java
@Controller
@RequestMapping("/basic/items")
@RequiredArgsConstructor
public class BasicItemController {

    private final ItemRepository itemRepository;
}
```
→ 롬복이 itemRepository의 생성자를 자동으로 생성해준다. <br>
→ 생성자가 1개만 있으면 스프링이 그 생성자에 `@Autowired`로 의존관계를 주입해준다.
<br>
<br>
<br>

> 정적 HTML으로 작성한 items.html을 뷰 템플릿 영역으로 복사해오고 타임리프를 적용해 코드를 수정하자!
#### templates/basic/items.html
```html
<html xmlns:th="http://wwww.thymeleaf.org">
<head>
    <meta charset="utf-8">
    <link th:href="@{/css/bootstrap.min.css}"
            href="../css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
<div class="container" style="max-width: 600px">
    <div class="py-5 text-center">
        <h2>상품 목록</h2></div>
    <div class="row">
        <div class="col">
            <button class="btn btn-primary float-end"
                    onclick="location.href='addForm.html'"
                    th:onclick="|location.href='@{/basic/items/add}'|"
                    type="button">상품 등록
            </button>
        </div>
    </div>
    <hr class="my-4">
    <div>
        <table class="table">
            <thead>
            <tr>
                <th>ID</th>
                <th>상품명</th>
                <th>가격</th>
                <th>수량</th>
            </tr>
            </thead>
            <tbody>
            <tr th:each="item : ${items}">
                <td><a href="item.html" th:href="@{/basic/items/{itemId}(itemId=${item.id})}" th:text="${item.id}">회원id</a></td>
                <td><a href="item.html" th:href="@{/basic/items/{itemId}(itemId=${item.id})}" th:text="${item.id}">상품명</a></td>
                <td th:text="${item.price}">10000</td>
                <td th:text="${item.quantity}">10</td>
            </tr>
            </tbody>
        </table>
    </div>
</div> <!-- /container -->
</body>
</html>
```
<br>

### 타임리프
#### 사용선언
```html
<html xmlns:th="http://www.thymeleaf.org">
```
<br>
<br>

#### 타임리프의 핵심
- th:xx가 붙은 부분은 서버사이드에서 렌더링되고, 기존의 것을 대체한다는 것이 핵심이다. 만약 th:xx가 붙어있지 않으면 기존 html의 xx 속성이 그대로<br>
  사용된다.
- HTML을 파일로 직접 열었을 때는 th:xx가 붙어있어도 웹 브라우저는 th 속성을 모르기 때문에 무시한다.
- 그렇기 때문에 HTML을 파일 보기를 유지하면서 템플릿 기능도 할 수 있다.
<br>
<br>

#### 속성 변경
> th:href
```html
th:href="@{/css/bootstrap.min.css}"
```
- href="value1"을 th:href="value2" 값으로 변경한다.
- 대부분의 HTML 속성을 th:xx로 변경할 수 있다.
- 타임리프 뷰 템플릿을 거치면 원래의 값을 th:xx로 변경한다. 만약 값이 없으면 새로 생성한다.
- HTML을 그대로 열어볼 때는 href 속성이 사용되지만 뷰 템플릿을 거치면 th:href의 값이 href를 대체하면서 동적으로 변경될 수 있다.
<br>
<br>

> th:onclick
```html
th:onclick="|location.href='@{/basic/items/add}'|"
```
<br>
<br>

#### 리터럴 대체 
> |...|
```html
<span th:text="|Welcome to out application, ${user.name}!|">
```
- 타임리프에서 문자와 표현식은 분리되어 있기 때문에 더해서 사용해야 하는데 리터럴 대체 문법을 사용하면 편리하게 더할 수 있다.
<br>
<br>

#### URL 링크 표현식
> @{...}
```html
th:href="@{/css/bootstrap.min.css}"
```
- URL 링크를 사용하는 경우 `@{...}`를 사용하며 이를 URL 링크 표현식이라고 한다.
- URL 링크 표현식을 사용하면 서블릿 컨텍스트를 자동으로 포함한다.(참고)
<br>
<br>

#### 반복 출력
> th:each
```html
<tr th:each="item : ${items}">
```
- 모델에 포함된 items 컬렉션의 데이터가 item 변수에 하나씩 포함되고, 반복문 안에서 item 변수를 사용할 수 있다.
- 컬렉션의 데이터 수만큼 <tr></tr>이 하위 태그를 포함해 생성된다.
<br>
<br>

#### 변수 표현식
> ${...}
```html
<td th:text="${item.price}">10000</td>
```
- 모델에 포함된 값이나 타임리프 변수로 선언한 값을 조회할 수 있다.
- 프로퍼티 접근법을 사용한다.
    - Ex) item.getPrice()
<br>
<br>

> th:text
- 값을 th:text의 값으로 변경한다.
- Ex) 10000을 ${item.price}의 값으로 변경한다.
<br>
<br>

#### 참고
> 타임리프는 순수 HTML 파일을 웹 브라우저에서 열어 내용을 확인할 수 있고, 서버를 통해 뷰 템플릿을 거치면 동적으로 변경된 결과를 확인할 수 있다. <br>
> 그런데 JSP 파일은 웹 브라우저에서 그냥 열면 JSP 소스코드와 HTML이 뒤섞여 정상적인 확인이 불가능하고 오직 서버를 통해서 JSP를 열어야 한다. <br>
> 이렇게 순수 HTML을 그대로 유지하면서 뷰 템플릿도 사용할 수 있는 타임리프의 특징을 네츄럴 템플릿 (natural templates)이라 한다.
<br>
<br>
<br>
<br>

## 상품 상세
### 상품 상세 컨트롤러
#### BasicItemController
> 코드 
```java
@GetMapping("/{itemId}")
public String item(@PathVariable Long itemId, Model model) {
    Item item = itemRepository.findById(itemId);
    model.addAttribute("item", item);
    
    return "basic/item";
}
```
- GetMapping으로 item()을 실행하도로 해주고, @PathVariable로 넘어온 상품ID를 이용해 상품을 조회하고 모델에 담은 후 뷰 템플릿을 호출한다.
<br>
<br>

### 상품 상세 뷰
#### templates/basic/item.html
```html
<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8">
    <link th:href="@{/css/bootstrap.min.css}"
            href="../css/bootstrap.min.css" rel="stylesheet">
    <style>
                 .container {
            max-width: 560px;
}
    </style>
</head>
<body>
<div class="container">
    <div class="py-5 text-center">
        <h2>상품 상세</h2></div>
    <div>
        <label for="itemId">상품 ID</label>
        <input type="text" id="itemId" name="itemId" class="form-control"
               value="1" th:value="${item.id}" readonly> <!-- value 속성을 th:value 속성으로 변경 -->
    </div>
    <div>
        <label for="itemName">상품명</label>
        <input type="text" id="itemName" name="itemName" class="form-control"
               value="상품A" th:value="${item.itemName}" readonly></div>
    <div>
        <label for="price">가격</label>
        <input type="text" id="price" name="price" class="form-control"
               value="10000" th:value="${item.price}" readonly>
    </div>
    <div>
        <label for="quantity">수량</label>
        <input type="text" id="quantity" name="quantity" class="form-control"
               value="10" th:value="${item.quantity}" readonly>
    </div>
    <hr class="my-4">
    <div class="row">
        <div class="col">
            <button class="w-100 btn btn-primary btn-lg"
                    onclick="location.href='editForm.html'"
                    th:onclick="|location.href='@{/basic/items/{itemId}/edit(itemId=${item.id})}'|"
                    type="button">상품 수정
            </button>
        </div>
        <div class="col">
            <button class="w-100 btn btn-secondary btn-lg"
                    onclick="location.href='items.html'"
                    th:onclick="|location.href='@{/basic/items}'|"
                    type="button">목록으로
            </button>
        </div>
    </div>
</div> <!-- /container -->
</body>
</html>
```
<br>
<br>
<br>
<br>

## 상품 등록 폼
#### BasicItemController
> 코드 추가
```java
@GetMapping("/add")
public String addForm() {
   return "basic/addForm";
}
```
- @GetMapping 추가하고 뷰 템플릿만 호출한다.
<br>
<br>

### 상품 등록 뷰
#### templates/basic/addForm.html
```html
<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8">
    <link th:href="@{/css/bootstrap.min.css}"
            href="../css/bootstrap.min.css" rel="stylesheet">
    <style>
          .container {
              max-width: 560px;
}
    </style>
</head>
<body>
<div class="container">
    <div class="py-5 text-center">
        <h2>상품 등록 폼</h2></div>
    <h4 class="mb-3">상품 입력</h4>

    <form action="item.html" th:action method="post"> <!-- 나중에 치환이 되면 action이 비게 되고, 그러면 그냥 현재 url(items/add)에다가 값을 넘기게 된다.(Post) -->
        <div>
            <label for="itemName">상품명</label>
            <input type="text" id="itemName" name="itemName" class="form-control" placeholder="이름을 입력하세요"></div>
        <div>
            <label for="price">가격</label>
            <input type="text" id="price" name="price" class="form-control" placeholder="가격을 입력하세요">
        </div>
        <div>
            <label for="quantity">수량</label>
            <input type="text" id="quantity" name="quantity" class="form-control" placeholder="수량을 입력하세요"></div>
        <hr class="my-4">
        <div class="row">
            <div class="col">
                <button class="w-100 btn btn-primary btn-lg" type="submit">상품 등록
                </button>
            </div>
            <div class="col">
                <button class="w-100 btn btn-secondary btn-lg"
                        onclick="location.href='items.html'"
                        th:onclick="|location.href='@{/basic/items}'|"
                        type="button">취소
                </button>
            </div>
        </div>
    </form>
</div> <!-- /container -->
</body>
</html>
```
- 상품 등록 폼의 URL과 실제 상품 등록을 처리하는 URL을 똑같이 설정하고 **HTTP 메소드로 두 기능을 구분한다.**
    - 상품 등록 폼 → **GET** /basic/items/add
    - 상품 등록 처리 → **POST** /basic/items/add
<br>
<br>
<br>
<br>

## 상품 등록 처리
> @ModelAttribute
- 상품 등록 폼에서 전달된 데이터로 실제 상품 등록 처리를 구현해보자!
- 상품 등록 폼은 **HTML Form 방식**으로 서버에 데이터를 전달한다.
```
content-type: application/x-www-form-urlencoded
메시지 바디에 쿼리 파라미터 형식으로 전달한다.(Ex. itemName=itemA&price=10000&quantity=10)
```
<br>
<br>

### @RequestParam
> 요청 파라미터 형식을 처리하기 위해 @RequestParam을 사용하자

#### addItemV1
> BasicItemController에 추가해 상품 등록 처리를 구현해보자!
```java
@PostMapping("/add") //같은 url로 오더라도 Get, Post로 add와 save 구분
public String addItemV1(@RequestParam String itemName,
                        @RequestParam int price,
                        @RequestParam Integer quantity,
                        Model model) {

    //파라미터가 들어오면 실제 Item 객체를 생성해 itemRepository를 통해 저장한 후 저장된 item을 모델에 담아 뷰테 전달한다.
    Item item = new Item();
    item.setItemName(itemName);
    item.setPrice(price);
    item.setQuantity(quantity);

    itemRepository.save(item);
    model.addAttribute("item", item);

    return "basic/item"; //item.html 뷰 템플릿 재활용
}
```
<br>
<br>
<br>

#### addItemV2
- @RequestParam으로 변수를 하나하나 받아 Item을 생성하는 과정은 너무 불편하므로, **@ModelAttribute**를 사용해 한 번에 처리하자!
```java
/**
 * ModelAttribute 버전
 * @ModelAttribute("item") Item item
 */
@PostMapping("/add")
public String addItemV2(@ModelAttribute("item") Item item) {

    // ModelAttribute가 이 부분을 자동으로 처리
    /*
    Item item = new Item();
    item.setItemName(itemName);
    item.setPrice(price);
    item.setQuantity(quantity);
    */

    //ModelAttribute는 1. item 객체를 만들어주고, 2. 내가 지정한 이름대로 (item) model에 담아준다.
    itemRepository.save(item);
    //model.addAttribute("item", item); //자동 추가. 생략 가능

    return "basic/item";
}
```
#### @ModelAttribute의 기능
```
1. 요청 파라미터 처리
- Item 객체를 생성하고 요청 파라미터의 값을 프로퍼티 접근법으로(set___) 입력해준다. 

2. Model 추가
- Model에 @ModelAttribute로 지정한 객체를 자동으로 넣어준다. <br>
→ model.addAttribute("item", item) 부분을 자동으로 처리해준다.
```
<br>
<br>

- 모델에 데이터를 담을 때, 이름은 `@ModelAttribute`에 지정한 name(value) 속성을 사용한다. 
- 만약 다음과 같이 @ModelAttribute의 이름을 다르게 지정하면 다른 이름으로 모델에 들어간다.
```
@ModelAttribute("hello") Item item → 이름을 hello 로 지정 
model.addAttribute("hello", item); → 모델에 hello 이름으로 저장
```
<br>
<br>
<br>

#### addItemV3
> ModelAttribute 이름 생략 버전
```java
/**
 * @ModelAttribute name 생략 가능
 * model.addAttribute(item); 자동 추가. 생략 가능
 * 생략시 model에 저장되는 name은 클래스명 첫글자만 소문자로 바꾼 이름으로 등록 Item -> item 
 */
@PostMapping("/add")
public String addItemV3(@ModelAttribute Item item) {
    itemRepository.save(item);
    return "basic/item";
}
```
- 위와 같이 `@ModelAttribute`의 이름을 생략할 수도 있다.
    - 생략하면 모델에 저장될 때 클래스의 첫글자만 소문자로 변경한 이름으로 등록된다.
<br>
<br>
<br>

#### addItemV4
> ModelAttribute 전체 생략
```java
/**
 * @ModelAttribute 자체 생략 가능
 * model.addAttribute(item) 자동 추가 
 */
@PostMapping("/add")
public String addItemV4(Item item) {
    itemRepository.save(item);
    return "basic/item";
}
```
- @ModelAttribute를 생략하는 것도 가능하다. 대상 객체는 모델에 자동으로 등록된다.
<br>
<br>
<br>
<br>

## 상품 수정
> 상품 수정 폼 컨트롤러
#### BasicItemController 추가
```java
@GetMapping("/{itemId}/edit")
public String editForm(@PathVariable Long itemId, Model model) {
    Item item = itemRepository.findById(itemId);
    model.addAttribute("item", item);
    
    return "basic/editForm";
}
```
- 수정에 필요한 정보를 조회하고 수정용 폼 뷰(editForm)를 호출한다.
<br>
<br>

### 상품 수정 폼 뷰
- 정적 HTML을 뷰 템플릿 영역으로 복사 후 수정해보자!
#### editForm.html
```html
<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8">
    <link th:href="@{/css/bootstrap.min.css}"
            href="../css/bootstrap.min.css" rel="stylesheet">
    <style>
        .container {
            max-width: 560px;
}
    </style>
</head>
<body>
<div class="container">
    <div class="py-5 text-center">
        <h2>상품 수정 폼</h2></div>
    <form action="item.html" th:action method="post">
        <div>
            <label for="id">상품 ID</label>
            <input type="text" id="id" name="id" class="form-control" value="1" th:value="${item.id}" readonly>
        </div>
        <div>
            <label for="itemName">상품명</label>
            <input type="text" id="itemName" name="itemName" class="form-control" value="상품A" th:value="${item.itemName}">
        </div>
        <div>
            <label for="price">가격</label>
            <input type="text" id="price" name="price" class="form-control" value="10000" th:value="${item.price}">
        </div>
        <div>
            <label for="quantity">수량</label>
            <input type="text" id="quantity" name="quantity" class="form-control" value="10" th:value="${item.quantity}">
        </div>
        <hr class="my-4">
        <div class="row">
            <div class="col">
                <button class="w-100 btn btn-primary btn-lg" type="submit">저장
                </button>
            </div>
            <div class="col">
                <button class="w-100 btn btn-secondary btn-lg"
                        onclick="location.href='item.html'"
                        th:onclick="|location.href='@{/basic/items/{itemId}(itemId=${item.id})}'|"
                        type="button">취소
                </button>
            </div>
        </div>
    </form>
</div> <!-- /container -->
</body>
</html>
```
<br>
<br>
<br>

### 상품 수정
> 수정 버튼 클릭 시 상품이 수정되도록 구현해보자!
```java
@PostMapping("/{itemId}/edit")
public String edit(@PathVariable Long itemId, @ModelAttribute Item item) {
    itemRepository.update(itemId, item);

    return "redirect:/basic/items/{itemId}";
}
```
```
GET /items/{itemId}/edit → 상품 수정 폼
POST /items/{itemId}/edit → 상품 수정 처리
```
- 같은 경로로 오더라도 HTTP 메소드에 따라(GET, POST) 구분한다.
<br>
<br>
<br>

### 리다이렉트
- 상품 수정 단계는 마지막에 뷰 템플릿을 호출하지 않고 상품 상세 화면으로 이동하도록 **리다이렉트**를 호출하자.
- `redirect:/...`으로 리다이렉트를 호출할 수 있다.
- 컨트롤러에 매핑된 @PathVariable 값은 redirect에도 사용할 수 있다.
    - redirect:/basic/items/{itemId}에서 {itemId}는 @PathVariable Long itemId 값을 그대로 사용한다.(치환)
<br>
<br>
<br>

## PRG 
> Post/Redirect/Get
- 현재 `상품 등록 처리 컨트롤러`(addItemV1~addItemV4)는 심각한 문제가 있다. 상품 등록을 완료하고 웹 브라우저의 새로고침을 할 때마다 중복 등록된다.

<img width="800" alt="스크린샷 2022-09-20 오후 12 51 44" src="https://user-images.githubusercontent.com/80838501/191163655-59a7897e-f69a-4c93-86bd-fb9fa5848d12.png">

- 웹 브라우저 새로 고침은 마지막에 서버에 전송한 데이터를 다시 전송하는 것이다.
- 상품 등록 폼에서 저장 클릭 시 `POST /add + 상품 데이터`를 서버로 전송하는데, 현 상태에서 새로 고침을 하면 마지막에 전송한 `POST /add + 상품 데이터`를 서버로 재전송하게 되는 것이다.
  그래서 데이터는 동일하고 ID만 다른 상품 데이터가 계속 생성된다.
<br>
<br>
<br>

이 문제를 **PRG**로 해결해보자!

<img width="846" alt="스크린샷 2022-09-20 오후 12 58 09" src="https://user-images.githubusercontent.com/80838501/191164538-1886cc64-cb62-494c-a1ff-16b5dbefa89c.png">

- 상품 저장 후에 뷰 템플릿으로 이동하지 말고 상품 상세 화면으로 리다이렉트해주어 새로 고침 문제를 해결하자.
- 웹 브라우저는 상품 저장 후 상품 상세 화면으로 리다이렉트되기 때문에, 가장 마지막에 호출된 내용이 상품 상세 화면인 `GET /items/{id}`가 된다. <br>
  따라서, 새로고침을 해도 상품 상세 화면으로 이동하기 때문에 상품이 중복 저장되는 문제가 해결되는 것이다.
<br>
<br>

#### BasicItemController
```java
@PostMapping("/{itemId}/edit")
public String edit(@PathVariable Long itemId, @ModelAttribute Item item) {
    itemRepository.update(itemId, item);

    return "redirect:/basic/items/{itemId}";
}
```
<br>
<br>
<br>
<br>

## RedirectAttributes
> 상품 저장 후 상품 상세 화면으로 리다이렉트는 되지만, 고객 입장에서 상품 저장이 잘 된 것인지 확신이 들지 않는다. <br>
> 저장이 잘 되었으면 상품 상세 화면에 "저장완료!" 메시지를 보여주도록 수정해보자!
<br>
<br>

#### BasicItemController
```java
@PostMapping("/add")
public String addItemV6(Item item, RedirectAttributes redirectAttributes) {
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true); //true면 저장이 잘 된 것. 쿼리 파라미터로 넘어간다.

    return "redirect:/basic/items/{itemId}"; //savedItem.getId()값 치환
}
```
- RedirectAttributes를 이용해 리다이렉트될 때 `status=true`값이 추가되도록 수정하자. 그리고 뷰 템플릿에서 status 값이 true면 "저장완료!" 메시지를 출력하자.
- 실행 시 리다이렉트 결과는 다음과 같다. `http://localhost:8080/basic/items/3?status=true` 
<br>
<br>

#### templates/basic/item.html
```html
<div class="container">
    <div class="py-5 text-center"> 
        <h2>상품 상세</h2>
    </div>
    
    <!-- 추가 -->
    <h2 th:if="${param.status}" th:text="'저장 완료!'"></h2>
</div>
```
- status값이 true면 텍스트를 보이게 한다.
