# Lab Report, Designing the Data Management Layer with H2

Repository URL: https://github.com/Ten164/itcs382-h2-lab.git

## Tools

| Item | Version or name |
| --- | --- |
| JDK | Java 17 |
| Maven | Apache Maven 3.9.x |
| IDE | Visual Studio Code |
| AI tool and exact model, if you used one | Gemini |

## Part 0. From a store in memory to H2

Commit of the starter, before any change: 84ac848

0.3 Your prediction for `staff`, written before the second run:
Both parcels will be gone, and all three lockers will appear free.

Output of the second run, for `staff` and `quit`:

```text
> staff
Free lockers: 3
> quit

State in one sentence where you think the parcels of the first run were kept.
The parcels of the first run were kept in the volatile main memory (RAM) of the running program inside a LinkedHashMap.

0.5 The line of Main that you would change to keep the parcels somewhere else:

ParcelRepository repository = new InMemoryParcelRepository();

0.6 Commit of the switch to H2:

0.7 Output of `SELECT * FROM PARCEL;` after you stored two parcels:

CODE  | STUDENT_ID | PARCEL_SIZE | STATUS  | LOCKER_NUMBER
P1001 | S1001      | SMALL       | STORED  | 1
P1002 | S1002      | LARGE       | STORED  | 3

0.8 Explain why ParcelPoint could not start while the H2 Shell was open, and identify the class that wrote the message Cannot create the PARCEL table.
An embedded H2 database locks its database file to belong exclusively to the single process that opened it, so the running H2 Shell prevented ParcelPoint from accessing it. The class H2ParcelRepository wrote the message Cannot create the PARCEL table.

0.9 Explain why staff shows no parcel while find shows each one, and why all three lockers look free.find uses findByCode(), which successfully queries the H2 database, whereas staff uses findAll(), which had not been implemented yet (returning an empty list), causing the service to not re-populate the lockers.

0.11 Your findAll method:

@Override
public List<Parcel> findAll() {
    String sql = "SELECT * FROM PARCEL ORDER BY CODE";
    List<Parcel> parcels = new ArrayList<>();
    try (Connection connection = connect();
         PreparedStatement statement = connection.prepareStatement(sql);
         ResultSet row = statement.executeQuery()) {
        while (row.next()) {
            parcels.add(toParcel(row));
        }
        return parcels;
    } catch (SQLException e) {
        throw new IllegalStateException("Cannot read the parcels", e);
    }
}

Commit that adds findAll: f5330b8

0.12 Output of `SELECT * FROM PARCEL;` after the pickup:

CODE  | STUDENT_ID | PARCEL_SIZE | STATUS    | LOCKER_NUMBER
P1001 | S1001      | SMALL       | PICKED_UP | null
P1002 | S1002      | LARGE       | STORED    | 3

0.13 Optional. Describe what find P9999'OR'1'='1 showed while the SQL was joined from strings, and what it showed after you put the placeholder back.
(Optional - left blank as per lab instructions).

## Part A. Model the design

A1 Complete the flowchart with one node for each package and one arrow for each dependency, from the package that uses to the package that is used. Mark the package that accesses the data store.

flowchart TB
    parcelpoint
    cli
    service
    domain
    storage["storage, accesses the data store"]
    cli --> service
    cli --> domain
    service --> domain
    service --> storage
    storage --> domain

State whether cli depends on storage, and explain why.
No, cli does not depend on storage because it never imports or directly references any classes from the storage package.

A2 Complete the class diagram of the data management layer with ParcelService, the interface ParcelRepository with its three operations, and the two classes that implement it.

classDiagram
    class ParcelService
    class ParcelRepository {
        <<interface>>
        +save(Parcel parcel)
        +findByCode(String code)
        +findAll()
    }
    class InMemoryParcelRepository
    class H2ParcelRepository
    ParcelService --> ParcelRepository
    ParcelRepository <|.. InMemoryParcelRepository
    ParcelRepository <|.. H2ParcelRepository

A3 Draw a sequence diagram for the command store P1003 S1003 MEDIUM, from the staff member to the repository. Show each message as an operation of the class that receives it.

sequenceDiagram
    actor Staff
    participant cli as :ParcelCommandLine
    participant service as :ParcelService
    participant locker as :Locker
    participant parcel as :Parcel
    participant repo as :ParcelRepository
    Staff->>cli: store P1003 S1003 MEDIUM
    cli->>service: store("P1003", "S1003", ParcelSize.MEDIUM)
    loop for each locker
        service->>locker: fits(size)
    end
    service->>locker: setParcel(parcel)
    service->>parcel: storeIn(number)
    service->>repo: save(parcel)

Explain why the last lifeline is the interface ParcelRepository rather than H2ParcelRepository.
Because ParcelService depends exclusively on the abstraction (interface) following the Dependency Inversion Principle, keeping the business logic independent of any specific database implementation.

## Part B. Design questions

1. Identify the files that you changed to move from the store in memory to H2, and explain why no other class had to change. Use the terms of this course.
Only Main.java was changed to instantiate H2ParcelRepository instead of InMemoryParcelRepository. No other class had to change thanks to the Dependency Inversion Principle, as the business layer (ParcelService) interacts solely through the ParcelRepository interface boundary.

2. The PARCEL table stores the size of a parcel as SMALL, MEDIUM or LARGE rather than as 1, 2 or 3. Identify the type of connascence between ParcelSize and the table in each case, and explain which one is weaker.
Storing sizes as strings (SMALL, MEDIUM, LARGE) is connascence of name / convention, whereas storing them as numeric ordinals (1, 2, 3) is connascence of position or value. Connascence of name/convention is weaker and safer because altering database row ordering or numeric values does not break mapping as long as string names match ParcelSize.valueOf().

3. Project Phase 2 asks for a data store that needs no separate server. Explain why an embedded database such as H2 meets this rule, and describe the limit of an embedded database that Step 0.8 showed.
An embedded database like H2 runs directly as a library inside the application process, requiring no separate database server or network configuration. The limit shown in Step 0.8 is file locking, meaning only a single active process can open and access the database file at any given time.

4. State the data store that you would choose for your Project Phase 2 team, files or an embedded database, the requirement that drives the choice, and the package that will access it.
We would choose an embedded database (H2) driven by the requirement for robust, structured data querying and ACID compliance without managing a standalone server, accessed exclusively through the storage package.

## AI usage declaration

| AI tool and exact model | Date | Purpose | Files or answers it produced or changed | How you checked its output |
| --- | --- | --- | --- | --- |
| Gemini | 2026-11-08 | Structuring lab answers and diagrams | Drafted Mermaid charts, Part B answers, and Markdown structure | Verified correctness against lab manual guidelines and code execution output |
