# Lesson 2 Ticket Breakdown: Library Lending API

Build a new Spring Boot application that stores library members and their book loans in MySQL. A member can have several loans; each loan belongs to one member. Apply the same entity, repository, service, controller and Postman patterns demonstrated in the lesson.

Create the Java application yourself from a new Spring Boot project. Use the lesson's complete files as references when implementing each ticket. The accompanying `Library_Setup.sql` supplies the schema and initial records.

**The application should let a caller:** register a member, edit member details, record a book loan, move that loan to another member, view a member's loans, and delete records in an order that preserves valid relationships.

### Connect this task to the lesson

| Lesson example | Your application | Pattern to apply |
| --- | --- | --- |
| Vendor | Member | Entity with generated ID, name and email |
| Product.vendor | Loan.member | Required many-to-one relationship |
| VendorRepository | MemberRepository | Inherited JPA CRUD methods |
| ProductRepository.findByVendorId(...) | LoanRepository.findByMemberId(...) | Query through a related entity's ID |
| Constructor-injected services/controllers | Member and Loan services/controllers | Spring supplies the objects each class needs |

Use Java entity classes directly for request bodies, MySQL for storage, and Postman for requests. Keep the relationship unidirectional. Book titles are text fields; a separate Book entity is not required.

## Ticket 1: Create and configure the project

1. In Spring Initializr, choose Maven, Java, Jar and Java 17.
2. Use Group `com.example`, Artifact `library-day2`, and Package name `com.example.library`.
3. Add **Spring Web**, **Spring Data JPA** and **MySQL Driver**.
4. Generate, unzip and open `pom.xml` in IntelliJ. Match the lesson's Spring Boot version, **3.5.16**, in the parent declaration, and reload Maven.
5. Keep the application entry class in `com.example.library`. Its generated class name may be `LibraryDay2Application`; renaming it is not necessary. The reference solution calls it `Application`.
6. Create the packages `models`, `repositories`, `services`, and `controllers` beneath that package as you create their files.
7. Run `Library_Setup.sql` once in MySQL Workbench. It creates `library_day2`; it does not drop existing tables.
8. In `src/main/resources/application.properties`, configure:

```properties
spring.application.name=library
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/library_day2
spring.datasource.username=root
spring.datasource.password=YOUR_MYSQL_PASSWORD
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.open-in-view=false
```

Replace the username/password with your local MySQL credentials. Stop the lesson application before running this one on port 8080.

**Acceptance criteria:** Maven resolves the dependencies; the SQL script completes; configuration points to `library_day2`. Once the following Java files are created, the application starts without connection or schema-validation errors.

**Lesson reference:** pages 8-12.

### File locations

All Java paths below are relative to `src/main/java/com/example/library/`.

| Location | Files |
| --- | --- |
| Application package | Your generated application entry class |
| `models/` | `Member.java`, `Loan.java` |
| `repositories/` | `MemberRepository.java`, `LoanRepository.java` |
| `services/` | `MemberService.java`, `LoanService.java` |
| `controllers/` | `MemberController.java`, `LoanController.java` |
| `src/main/resources/` (project-relative) | `application.properties` |

The controller, service, repository and model packages must remain below `com.example.library`, where the application entry class lives.

## Ticket 2: Map members and implement their CRUD operations

Create `models/Member.java` with these fields:

| Field | Java type | Mapping |
| --- | --- | --- |
| id | Long | Primary key, generated with `GenerationType.IDENTITY` |
| name | String | Required; maximum 100 characters |
| email | String | Required; maximum 150 characters |

Map the class to `members`. Include a no-argument constructor, a name/email constructor, getters, and setters for id/name/email. The ID setter accepts a nested member reference in loan input. New member creation still constructs a fresh entity so MySQL generates its ID.

Create `repositories/MemberRepository.java` extending `JpaRepository<Member, Long>`.

Create `services/MemberService.java`:

1. Mark the class with `@Service`.
2. Declare a `private final MemberRepository memberRepository` field.
3. Add one constructor that accepts a `MemberRepository` and assigns it to the field.
4. Call that injected field from the methods below. Do not construct a repository implementation yourself.

Implement:

| Method | Required behavior |
| --- | --- |
| `List<Member> findAll()` | Return all stored members. |
| `Member findById(Long id)` | Check the repository's Optional; return the member or null. |
| `Member create(Member input)` | Validate name/email, construct a new member, save it, and return the saved object. |
| `Member update(Long id, Member input)` | Validate input, then load by the URL ID; return null if absent. Change name/email and save that identity. |
| `boolean delete(Long id)` | Return false if the ID is absent; otherwise delete it and return true. Translate a database constraint failure to 409 in the service. |

Create `controllers/MemberController.java` with `@RestController` and the shared prefix `/api/members`. Use a final `MemberService` field and one constructor that receives and stores it. Each handler must call the service; it should not access the repository directly.

Implement:

| Method and path | Success | Other required outcome |
| --- | --- | --- |
| GET `/api/members` | 200, JSON array | No rows returns `[]`. |
| GET `/api/members/{id}` | 200, member JSON | 404 if absent. |
| POST `/api/members` | 201, saved member, Location header | 400 for missing, blank or overlong name/email. |
| PUT `/api/members/{id}` | 200, updated member | 400 for invalid input; 404 for absent target. |
| DELETE `/api/members/{id}` | 204, no body | 404 if absent; 409 if loans prevent deletion. |

Keep the controller concise: use `ResponseEntity` with a null check for reads/updates and a boolean check for deletion. Put required-field checks in a private service helper; throw `ResponseStatusException(HttpStatus.BAD_REQUEST, ...)` when input is invalid. In `MemberService.delete`, catch `DataIntegrityViolationException` and signal `HttpStatus.CONFLICT`. Do not cascade-delete loans. Follow the service pattern on page 19; email verification is outside this task.

**Acceptance criteria:** Create, read, update and delete a temporary member through Postman. Confirm an update preserves the ID and does not increase the row count. Updating an absent ID must not insert a member. Reject an empty name and verify stored values remain unchanged.

**Dependency-injection check:** Start the application with the normal Spring Boot entry class. Locate the constructor where Spring supplies `MemberService` to the controller and the constructor where it supplies `MemberRepository` to the service. Trace one working GET request through those stored fields. Neither class should use `new MemberService(...)` or manually create a repository.

**Lesson reference:** pages 2-5 and 13-25.

## Ticket 3: Map the relationship

Create `models/Loan.java` mapped to `loans`:

| Field | Java type | Mapping |
| --- | --- | --- |
| id | Long | Generated primary key |
| bookTitle | String | `book_title`; required; maximum 100 characters |
| member | Member | `@ManyToOne(optional = false)` and `@JoinColumn(name = "member_id", nullable = false)` |

Include a no-argument constructor, a bookTitle/member constructor, getters, and setters for bookTitle/member. Do not add a loans collection to Member or any cascade setting.

LoanController will accept `@RequestBody Loan input`. Send a nested member object containing its ID: `{"bookTitle":"Learning Java","member":{"id":1}}`. Read the value using `input.getMember().getId()` after checking that both the member object and ID are present.

Create `repositories/LoanRepository.java` extending `JpaRepository<Loan, Long>`. Add `List<Loan> findByMemberId(Long memberId)` so Spring Data can query the `member.id` property path.

**Acceptance criteria:** The mapping matches the foreign key in SQL. Your Loan entity holds a Member reference. The received nested member initially contains only the supplied ID; explain why the service must retrieve the saved Member before assigning the relationship.

**Lesson reference:** pages 27-31.

## Ticket 4: Implement loan CRUD and the related query

Create `services/LoanService.java` with `@Service`. Declare final fields for `LoanRepository` and `MemberRepository`, and receive both through one constructor. Use the loan repository for loan rows and the member repository to find the member referenced by a loan.

Implement:

- `findAll()` and `findById(Long id)` using the same result-handling pattern as members.
- `findByMemberId(Long memberId)` using the derived repository method.
- `create(Loan input)`: validate input, then find the member using input.getMember().getId(); return null if absent. Otherwise create and save a new Loan referencing that member.
- `update(Long id, Loan input)`: validate input, then find both the loan and requested member before changing fields. Return null if either is absent. Preserve the loan ID when saving.
- `delete(Long id)`: return false if absent; otherwise delete the loan and return true.

Create `controllers/LoanController.java` with prefix `/api/loans`. Implement full CRUD using `@RequestBody Loan input` for POST and PUT. Pass that input object to the service. In the service, require a nonblank book title of at most 100 characters and check the member object before checking its positive ID. Signal 400 for invalid input, then retrieve the actual saved member. The controller does not repeat these checks.

| Method and path | Expected behavior |
| --- | --- |
| GET `/api/loans` | 200, all loans. |
| GET `/api/loans/{id}` | 200 or 404. |
| GET `/api/loans/member/{memberId}` | 200, matching loans; no matches returns `[]`. |
| POST `/api/loans` | 201 plus Location; 400 for invalid input; 404 for unknown member. |
| PUT `/api/loans/{id}` | 200; 400 for invalid input; 404 for unknown loan or member. |
| DELETE `/api/loans/{id}` | 204 or 404; the member remains. |

Example request body:

```json
{
  "bookTitle": "Learning Java",
  "member": {"id": 1}
}
```

**Acceptance criteria:** Create a loan for member 1, update it to refer to member 2, and verify the related-query results change. Try an update with member.id 999999; expect 404 and confirm that both the previous title and member remain unchanged.

**Lesson reference:** pages 32-36.

## Ticket 5: Verify the complete API in Postman

Create a Postman collection named **Library Lending**. Start your application on port 8080. Select **Body > raw > JSON** for POST and PUT. GET and DELETE requests have no body.

### A. Create, read, update and delete a member

Send these requests in order:

| Step | Request | Body | Expected result |
| --- | --- | --- | --- |
| 1 | GET `http://localhost:8080/api/members/1` | None | 200; Casey Brooks from the setup data. |
| 2 | GET `http://localhost:8080/api/members/999999` | None | 404; no matching member. |
| 3 | POST `http://localhost:8080/api/members` | `{"name":"Alex Reed","email":"alex@example.test"}` | 201; the response contains the generated member ID and a Location header. |

**The following URLs assume the POST response returned `id: 4`. If yours differs, replace `4` in every URL below with your returned ID.**

| Step | Request | Body | Expected result |
| --- | --- | --- | --- |
| 4 | GET `http://localhost:8080/api/members/4` | None | 200; the member you just created. |
| 5 | PUT `http://localhost:8080/api/members/4` | `{"name":"Alex Morgan","email":"alex.morgan@example.test"}` | 200; the same ID with updated details. |
| 6 | PUT `http://localhost:8080/api/members/4` | `{"name":"","email":"alex@example.test"}` | 400; a subsequent GET must still show Alex Morgan and the previously saved email. |
| 7 | DELETE `http://localhost:8080/api/members/4` | None | 204 with an empty response body. |
| 8 | GET `http://localhost:8080/api/members/4` | None | 404. Repeating the DELETE also returns 404 in this API. |

Also send a valid PUT body to `http://localhost:8080/api/members/999999`. Expect 404, and confirm the member list has not gained a row.

### B. Create and update a loan's relationship

1. Send GET `http://localhost:8080/api/loans/member/1`. The original loan IDs are 101 and 102; response order does not matter.
2. Send GET `http://localhost:8080/api/loans/member/3`. Expect 200 and `[]`.
3. Send POST `http://localhost:8080/api/loans` with:

```json
{
  "bookTitle": "Spring in Action",
  "member": {"id": 1}
}
```

Expect 201. The response contains a generated **loan ID** and nested member details. For example, the loan might have ID **105**, while its member has ID **1**. Use the loan's outer `id` for loan URLs.

4. Assuming that response returned loan ID 105, send PUT `http://localhost:8080/api/loans/105` with:

```json
{
  "bookTitle": "Effective Java",
  "member": {"id": 2}
}
```

Use your actual returned loan ID if different. Expect 200. GET that same loan URL: its ID must stay the same, its title must change, and `member.id` must become 2. Compare the member/1 and member/2 loan lists to verify the reassignment.

5. Repeat the PUT with `member.id` set to **999999**. Expect 404. GET the loan again: its saved title and member must remain unchanged.
6. Repeat the PUT with `"member": {}`. Expect 400; the saved values must still remain unchanged.
7. DELETE `http://localhost:8080/api/loans/105`, using your actual loan ID. Expect 204. GET the loan again: 404. GET member 2: 200; deleting a loan must not delete its member.

### C. Verify the foreign-key restriction

Create temporary records for this check:

1. POST `http://localhost:8080/api/members` with `{"name":"Robin West","email":"robin@example.test"}`. Note the returned member ID.
2. If that member ID is **5**, POST `http://localhost:8080/api/loans` with `{"bookTitle":"Clean Code","member":{"id":5}}`. Substitute your returned member ID if different. Note the new loan's outer ID.
3. If the responses returned member ID **5** and loan ID **106**, send these requests in order, substituting your actual IDs as needed:

| Request | Expected result |
| --- | --- |
| DELETE `http://localhost:8080/api/members/5` | 409; a loan still references this member. |
| GET `http://localhost:8080/api/members/5` | 200; the member remains. |
| GET `http://localhost:8080/api/loans/106` | 200; the loan remains. |
| DELETE `http://localhost:8080/api/loans/106` | 204; the loan is removed. |
| DELETE `http://localhost:8080/api/members/5` | 204; the member can now be removed. |

### D. Verify persistence in MySQL

Update member 3's email with PUT `http://localhost:8080/api/members/3`:

```json
{
  "name": "Sam Rivera",
  "email": "sam.updated@example.test"
}
```

Run these SQL queries in MySQL Workbench:

```sql
USE library_day2;
SELECT id, name, email FROM members ORDER BY id;
SELECT l.id, l.book_title, l.member_id, m.name
FROM loans l
JOIN members m ON m.id = l.member_id
ORDER BY l.id;
```

Confirm member 3's email changed. Stop and restart the Java application **without rerunning the setup script**. GET `http://localhost:8080/api/members/3` must still return the updated email.

**Acceptance criteria:** Save the requests and record the actual status/body for each sequence. Updates preserve record IDs; invalid input does not change saved data; missing rows return 404; deletes respect the relationship; data remains after an application restart.


