### 1. JDBC 이해

예전에는 db를 변경하면 애플리케이션 코드를 그에 맞게 다 변경해야 했다.
db와 연결하고, SQL을 전달하고, 데이터를 받아오는 부분을 모두 코드로 구현해야 했기 떄문이다.

jdbc에서는 db와 통신할 때 사용하는 인터페이스를 추상화해두었다.
각각의 db 벤더들은 자신의 db에 접근할 수 있는 jdbc 드라이버를 개발해서 제공한다.
해당 드라이버는 모두 jdbc 인터페이스에 맞게 구현되어 있다

이렇게 바뀌면서 개발자는 jdbc 표준 api를 사용해서 개발하기만 하면 되도록 바뀌었다.
만약 사용하는 db 종류가 바뀐다면 드라이버만 바꿔서 끼워주면 된다.

### 2. JDBC와 최신 데이터 접근 기술

대표적인 데이터 접근 기술에는 ObjectMapper와 JPA가 있다.
ObjectMapper의 경우에는 sql을 전달하면 그 후의 처리(받아온 데이터를 객체에 넣는 작업)는 알아서 이루어진다는 장점이 있다.
하지만 sql을 직접 작성해야 한다는 단점이 있다.
Jpa는 sql 작성 없이 객체만 전달하면 sql 짜는 것부터 알아서 해주는 편리한 기술이다.
하지만 그 자체로 복잡한 기술이여서, 제대로 쓰려면 학습이 필요하다.

기억해야 할 점은, 어떤 기술을 사용하든 그 기반에서는 결국 jdbc를 사용한다는 것이다.
따라서 간단하게라도 jdbc를 사용하는 방법을 익힐 필요가 있다

### 3. 데이터베이스 연결

어플리케이션에는 다양한 jdbc driver가 등록될 수 있다.
DriverManager.getConnection()는 등록된 드라이버 중 적절한 드라이버를 찾아서 db에 연결을 맺게 해준다.

```java
Connection connection = DriverManager.getConnection(URL, USERNAME,
PASSWORD);
```

이 때 모든 드라이버를 순회하면서 db url이 각 드라이버에서 처리할 수 있는 형식인지를 확인 후, 처리 가능한 드라이버를 통해 db와 연결을 맺는다.
이렇게 선택된 드라이버에서 구현한 java sql Connection 객체가 반환된다.

### 4. JDBC 개발 - 등록

실행할 sql문을 스트링으로 작성하고, 이를 Statement 생성자에 넘겨서 객체를 생성한다.
앞서 연결한 Connection 객체로부터 Statement 객체를 생성할 수 있다.

```java
String sql = "insert into member(member_id, money) values(?, ?)";

Connection con = getConnection();
PreparedStatement pstmt = con.prepareStatement(sql);
```

PreparedStatement를 사용하면 원하는 변수를 바인딩할 수 있다. (sql injection을 막기 위한 것)

executeUpdate()를 호출하면 입력한 update문이 실행되고, 그 결과가 반환된다.

```java
pstmt.executeUpdate();
```

모든 db 실행을 마친 후에는 connection을 닫는 것을 잊지 말아야 한다. 생성된 역순으로 statement, connection 순서로 닫아준다

```
pstmt.close();
con.close();
```

(위 코드에서는 에러 처리를 제외해서 간단하게 작성했으나, 실제 코드에서는 에러 핸들링이 필요하다)

### 5. JDBC 개발 - 조회

조회도 등록과 대부분의 플로우가 비슷하다.

```java
String sql = "select * from member where member_id = ?";

Connection con = getConnection();
PreparedStatement pstmt = con.prepareStatement(sql);
ResultSet rs = pstmt.executeQuery();
```

조회의 경우 쿼리를 실행한 결과를 ResultSet 객체로 받는다.  
resultSet.next()를 호출하면 커서가 한 칸씩 이동하면서 모든 데이터를 순회할 수 있다.
이렇게 이동한 데이터로부터 각 컬럼 값을 꺼내고, 새로운 엔티티 객체에 해당 값을 넣어준다

```java
if (rs.next()) {
  Member member = new Member();
  member.setMemberId(rs.getString("member_id"));
  member.setMoney(rs.getInt("money"));
  return member;
}
```

### 6. JDBC 개발 - 수정, 삭제

수정, 삭제의 경우도 앞선 생성/조회와 동일하게 ?가 포함되어 있는 쿼리로 만든 PreparedStatement를 이용하여, 변수값을 바인딩해서 실행시키면 된다. 생성의 경우와 동일하게 executeUpdate를 호출하면 된다.
