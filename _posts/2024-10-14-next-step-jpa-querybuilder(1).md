---
title: QueryBuilder DDL 구현
author: SangkiHan
date: 2024-10-14 11:20:00 +0900
categories: [NEXT-STEP, JPA]
tags: [NEXT-STEP]
---
------------

[Github Repo](https://github.com/SangkiHan/jpa-query-builder)


## DDL이란
DDL은 Data Definition Language의 약자로, 데이터베이스의 구조와 스키마를 정의하고 변경하는 SQL 명령어들을 포함하는 언어이다. DDL은 데이터베이스에서 테이블, 인덱스, 뷰 등의 객체를 생성, 수정, 삭제하기 위한 명령어를 제공한다.

주요 DDL 명령어는 다음과 같다

+ CREATE: 새로운 데이터베이스 객체(예: 테이블, 뷰 등)를 생성합니다.  
+ ALTER: 기존의 데이터베이스 객체를 수정합니다. 예를 들어, 테이블에 새로운 컬럼을 추가할 수 있습니다.  
+ DROP: 데이터베이스 객체를 삭제합니다. 이 명령어를 사용할 경우 해당 객체에 저장된 모든 데이터도 함께 삭제됩니다.  
+ TRUNCATE: 테이블의 모든 데이터는 삭제하지만, 테이블 구조는 유지합니다. 이 경우 일반적으로 더 빠르게 실행됩니다.

이번과정에는 Entity를 입력받아 CREATE, DROP 쿼리를 구현한다.

### ColumnData
ColumnData 객체를 생성하여 Entity의 각 컬럼 데이터를 저장하도록 하였다.
``` java
public class ColumnData {

    private String columnName;
    private String columnDataType;
    private boolean isPrimaryKey;
    private boolean isNotNull;
    private boolean isAutoIncrement;

    public String getColumnName() {
        return columnName;
    }

    public String getColumnDataType() {
        return columnDataType;
    }

    public boolean isPrimaryKey() {
        return isPrimaryKey;
    }

    public boolean isNotNull() {
        return isNotNull;
    }

    public boolean isAutoIncrement() {
        return isAutoIncrement;
    }

    //PK 컬럼을 생성한다.
    public void createPk(String columnName, Class<?> columnDataType, boolean isAutoIncrement) {
        this.columnName = columnName;
        this.columnDataType = H2DataType.findH2DataTypeByDataType(columnDataType);
        this.isPrimaryKey = true;
        this.isNotNull = true;
        this.isAutoIncrement = isAutoIncrement;
    }

    //일반 컬럼을 생성한다.
    public void createColumn(String columnName, Class<?> columnDataType, boolean isNotNull) {
        this.columnName = columnName;
        this.columnDataType = H2DataType.findH2DataTypeByDataType(columnDataType);
        this.isPrimaryKey = false;
        this.isNotNull = isNotNull;
        this.isAutoIncrement = false;
    }
}
```

### H2DataType

DB마다 데이터타입이 다르기 때문에 이번 실습에서 사용되는 H2 Database에 맞는 Enum을 생성하여 변수타입에 다른 DB데이터타입을 가져오도록 생성하였다.

``` java
public enum H2DataType {
    STRING(String.class, "VARCHAR(255)"),
    INTEGER(Integer.class, "INTEGER"),
    LONG(Long.class, "BIGINT");

    private final Class<?> dataType;
    private final String h2DataType;

    private final static String NOT_ALLOWED_DATATYPE = "지원하지 않은 데이터타입입니다. DataType: ";

    H2DataType(Class<?> dataType, String h2DataType) {
        this.dataType = dataType;
        this.h2DataType = h2DataType;
    }

    public Class<?> getDataType() {
        return dataType;
    }

    public String getH2DataType() {
        return h2DataType;
    }

    // dataType으로 H2DataType을 찾고 반환하는 메소드
    public static String findH2DataTypeByDataType(Class<?> dataType) {
        return Arrays.stream(values())
                .filter(type -> type.getDataType().equals(dataType))
                .map(H2DataType::getH2DataType)
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException(NOT_ALLOWED_DATATYPE + dataType));
    }
}
```

### H2QueryBuilderDDL

어떤 DB가 들어와도 create, drop쿼리 생성 기능은 필수적으로 존재 하기 때문에 interface를 생성하여 DB에 따라 유연하게 동작하도록 하였다.

``` java
public interface QueryBuilderDDL {

    //create쿼리를 생성한다.
    String buildCreateQuery(Class<?> entityClass);

    //drop 쿼리를 생성한다.
    String buildDropQuery(Class<?> entityClass);

}
```

``` java
public class H2QueryBuilderDDL implements QueryBuilderDDL {

    private final static String NOT_EXIST_ENTITY_ANNOTATION = "@Entity 어노테이션이 존재하지 않습니다.";
    private final static String ID_ANNOTATION_OVER_ONE = "@Id 어노테이션은 한개를 초과할수 없습니다.";
    private final static String CREATE_QUERY = "CREATE TABLE {tableName} ({columnDefinitions});";
    private final static String DROP_QUERY = "DROP TABLE {tableName};";
    private final static String PRIMARY_KEY = " PRIMARY KEY";
    private final static String NOT_NULL = " NOT NULL";
    private final static String AUTO_INCREMENT = " AUTO_INCREMENT";
    private final static String COMMA = ", ";
    private final static String BLANK = " ";
    private final static String TABLE_NAME = "{tableName}";
    private final static String COLUMN_DEFINITIONS = "{columnDefinitions}";

    //create 쿼리를 생성한다.
    @Override
    public String buildCreateQuery(Class<?> entityClass) {
        confirmEntityAnnotation(entityClass);
        return createTableQuery(getTableName(entityClass), getColumnData(entityClass));
    }

    //drop 쿼리를 생성한다.
    @Override
    public String buildDropQuery(Class<?> entityClass) {
        confirmEntityAnnotation(entityClass);
        return dropTableQuery(getTableName(entityClass));
    }

    //create 쿼리를 생성한다.
    public String createTableQuery(String tableName, List<ColumnData> columns) {
        // 테이블 열 정의 생성
        String columnDefinitions = columns.stream()
                .map(column -> {
                    String definition = column.getColumnName() + BLANK + column.getColumnDataType();
                    // primary key인 경우 "PRIMARY KEY" 추가
                    if (column.isNotNull()) definition += NOT_NULL; //false면 NOT_NULL 조건 추가
                    if (column.isAutoIncrement()) definition += AUTO_INCREMENT; //true면 AutoIncrement 추가
                    if (column.isPrimaryKey()) definition += PRIMARY_KEY; //PK면 PK조건 추가
                    return definition;
                })
                .collect(Collectors.joining(COMMA));

        // 최종 SQL 쿼리 생성
        return CREATE_QUERY.replace(TABLE_NAME, tableName)
                .replace(COLUMN_DEFINITIONS, columnDefinitions);
    }

    //Drop 쿼리 생성
    public String dropTableQuery(String tableName) {
        return DROP_QUERY.replace(TABLE_NAME, tableName);
    }

    //Entity 어노테이션 여부를 확인한다.
    private void confirmEntityAnnotation(Class<?> entityClass) {
        if (!entityClass.isAnnotationPresent(Entity.class)) {
            throw new IllegalArgumentException(NOT_EXIST_ENTITY_ANNOTATION);
        }
    }

    //Table 어노테이션 여부를 확인한다.
    private String getTableName(Class<?> entityClass) {
        if (entityClass.isAnnotationPresent(Table.class)) {
            Table table = entityClass.getAnnotation(Table.class);
            return table.name();
        }
        return entityClass.getSimpleName();
    }

    //변수들의 정보를 가져온다.
    private List<ColumnData> getColumnData(Class<?> entityClass) {
        Field[] fields = entityClass.getDeclaredFields();
        List<ColumnData> columnDataList = new ArrayList<>();

        for (Field field : fields) {
            createTableColumnData(columnDataList, field);
        }

        return columnDataList;
    }

    //테이블에 생성될 필드(컬럼)들을 생성한다.
    private void createTableColumnData(List<ColumnData> columnDataList, Field field) {
        getPrimaryKey(columnDataList, field);
        getColumnAnnotationData(columnDataList, field);
    }

    //Id 어노테이션을 primarykey로 가져온다.
    private void getPrimaryKey(List<ColumnData> columnDataList, Field field) {
        if (field.isAnnotationPresent(Id.class)) {
            confirmIdAnnotationOverTwo(columnDataList);
            ColumnData columnData = new ColumnData();
            columnData.createPk(field.getName(), field.getType(), confirmGeneratedValueAnnotation(field));
            columnDataList.add(columnData);
        }
    }

    //Column 어노테이션 여부를 확인하여 변수의 컬럼타입을 가져온다.
    private void getColumnAnnotationData(List<ColumnData> columnDataList, Field field) {
        if (field.isAnnotationPresent(Transient.class) || field.isAnnotationPresent(Id.class))
            return; // Transient 어노테이션이 있거나 @Id인 경우 검증하지 않음

        String columnName = field.getName();
        boolean isNullable = true;

        if (field.isAnnotationPresent(Column.class)) {
            Column column = field.getAnnotation(Column.class);
            columnName = column.name().isEmpty() ? columnName : column.name();
            isNullable = column.nullable();
        }

        ColumnData columnData = new ColumnData();
        columnData.createColumn(columnName, field.getType(), !isNullable);
        columnDataList.add(columnData);
    }

    //Entity에 @Id가 2개 이상은 아닐지 확인한다.
    private void confirmIdAnnotationOverTwo(List<ColumnData> columnDataList) {
        boolean hasPrimaryKey = columnDataList.stream()
                .anyMatch(ColumnData::isPrimaryKey); // primaryKey가 true인 컬럼이 하나라도 있는지 확인

        if (hasPrimaryKey) {
            throw new IllegalArgumentException(ID_ANNOTATION_OVER_ONE);
        }
    }

    //GeneratedValue 어노테이션 전략을 확인한다.
    private boolean confirmGeneratedValueAnnotation(Field field) {
        if (!field.isAnnotationPresent(GeneratedValue.class)) {
            return false;
        }
        GeneratedValue generatedValue = field.getAnnotation(GeneratedValue.class);
        return generatedValue.strategy() == GenerationType.IDENTITY;
    }

}
```

### Junit Test

#### H2DataType Enum 테스트
``` java
public class H2DataTypeTest {

    @DisplayName("변수 데이터타입에 따른 H2 컬럼 데이터타입을 가져온다.")
    @ParameterizedTest
    @CsvSource(value = {"java.lang.String:VARCHAR(255)", "java.lang.Integer:INTEGER", "java.lang.Long:BIGINT"}, delimiter = ':')
    void getDataTypeTest(String dataType, String h2DataType) {
        assertThat(H2DataType.findH2DataTypeByDataType(getClassForName(dataType))).isEqualTo(h2DataType);
    }

    private Class<?> getClassForName(String className) {
        try {
            return Class.forName(className);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException("존재하지 않는 클래스입니다.");
        }
    }
}
```

#### H2QueryBuilderDDL 테스트

Person Entity에 따라 검증이 달라지기 때문에 InnerClass로 테스트코드 내부에 Person을 선언하여 테스트하였다.

``` java
public class H2QueryBuilderDDLTest {
    @DisplayName("create 쿼리 생성시 Entity어노테이션이 존재하지 않으면 예외를 발생시킨다.")
    @Test
    void buildCreateQueryNotExistEntityThrowException() {
        //given
        class Person {
            @Id
            @GeneratedValue(strategy = GenerationType.IDENTITY)
            private Long id;

            @Column(name = "nick_name")
            private String name;

            @Column(name = "old")
            private Integer age;

            @Column(nullable = false)
            private String email;

        }
        H2QueryBuilderDDL h2QueryBuilderDDL = new H2QueryBuilderDDL();

        //when, then
        assertThatThrownBy(() -> h2QueryBuilderDDL.buildCreateQuery(Person.class))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessage("@Entity 어노테이션이 존재하지 않습니다.");
                }

    @DisplayName("create쿼리를 생성 시 @Id가 지정된 변수를 PK로 가져간다.")
    @Test
    void confirmIdAnnotationTest() {
        //given
        @Entity
        class Person {

            @Id
            private Long id;

        }
        H2QueryBuilderDDL h2QueryBuilderDDL = new H2QueryBuilderDDL();

        //when
        String createQuery = h2QueryBuilderDDL.buildCreateQuery(Person.class);

        //then
        assertThat(createQuery).isEqualTo(
                "CREATE TABLE Person (id BIGINT NOT NULL PRIMARY KEY);"
        );
    }

    @DisplayName("create쿼리를 생성 시 @Column이 지정되어있지 않으면 변수명을 컬럼명으로 생성한다.")
    @Test
    void notExistColumnAnnotationTest() {
        //given
        @Entity
        class Person {

            @Id
            private Long id;

            private String name;

            private Integer age;

            private String email;

        }
        H2QueryBuilderDDL h2QueryBuilderDDL = new H2QueryBuilderDDL();

        //when
        assertThat(h2QueryBuilderDDL.buildCreateQuery(Person.class)).isEqualTo(
                "CREATE TABLE Person (id BIGINT NOT NULL PRIMARY KEY, name VARCHAR(255), age INTEGER, email VARCHAR(255));"
        );
    }

    @DisplayName("create쿼리를 생성 시 @Column이 지정되어 있으면 확인하여 생성한다.")
    @Test
    void existColumnAnnotationTest() {
        //given
        @Entity
        class Person {

            @Id
            private Long id;

            @Column(name = "nick_name")
            private String name;

            @Column(name = "old")
            private Integer age;

            @Column(nullable = false)
            private String email;

        }
        H2QueryBuilderDDL h2QueryBuilderDDL = new H2QueryBuilderDDL();

        //when, then
        assertThat(h2QueryBuilderDDL.buildCreateQuery(Person.class)).isEqualTo(
                "CREATE TABLE Person (id BIGINT NOT NULL PRIMARY KEY, nick_name VARCHAR(255), old INTEGER, email VARCHAR(255) NOT NULL);"
        );
    }

    @DisplayName("create쿼리를 생성 시 @Id가 2개 이상이면 예외가 발생한다.")
    @Test
    void existIdAnnotationOverTwoTest() {
        //given
        @Entity
        class Person {

            @Id
            private Long id;

            @Id
            private String name;

        }
        H2QueryBuilderDDL h2QueryBuilderDDL = new H2QueryBuilderDDL();

        //when, then
        assertThatThrownBy(() -> h2QueryBuilderDDL.buildCreateQuery(Person.class))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessage("@Id 어노테이션은 한개를 초과할수 없습니다.");
    }

    @DisplayName("create쿼리를 생성 시 @Table이 지정되어있다면 테이블명을 가져온다.")
    @Test
    void existTableAnnotationOverTwoTest() {
        //given
        @Table(name = "users")
        @Entity
        class Person {

            @Id
            private Long id;

            @Column(name = "nick_name")
            private String name;

            @Column(name = "old")
            private Integer age;

            @Column(nullable = false)
            private String email;

        }
        H2QueryBuilderDDL h2QueryBuilderDDL = new H2QueryBuilderDDL();

        //when, then
        assertThat(h2QueryBuilderDDL.buildCreateQuery(Person.class)).isEqualTo(
                "CREATE TABLE users (id BIGINT NOT NULL PRIMARY KEY, nick_name VARCHAR(255), old INTEGER, email VARCHAR(255) NOT NULL);"
        );
    }

    @DisplayName("create쿼리를 생성 시 @GeneratedValue가 지정되어있다면 AUTOINCREMENT을 추가한다.")
    @Test
    void existGeneratedValueAnnotationOverTwoTest() {
        //given
        @Table(name = "users")
        @Entity
        class Person {

            @Id
            @GeneratedValue(strategy = GenerationType.IDENTITY)
            private Long id;

            @Column(name = "nick_name")
            private String name;

            @Column(name = "old")
            private Integer age;

            @Column(nullable = false)
            private String email;

        }
        H2QueryBuilderDDL h2QueryBuilderDDL = new H2QueryBuilderDDL();

        //when, then
        assertThat(h2QueryBuilderDDL.buildCreateQuery(Person.class)).isEqualTo(
                "CREATE TABLE users (id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY, nick_name VARCHAR(255), old INTEGER, email VARCHAR(255) NOT NULL);"
        );
    }

    @DisplayName("create쿼리를 생성 시 @Transient가 지정되어있다면 컬럼을 생성하지 않는다.")
    @Test
    void existTransientAnnotationOverTwoTest() {
        //given
        @Table(name = "users")
        @Entity
        class Person {

            @Id
            @GeneratedValue(strategy = GenerationType.IDENTITY)
            private Long id;

            @Column(name = "nick_name")
            private String name;

            @Column(name = "old")
            private Integer age;

            @Column(nullable = false)
            private String email;

            @Transient
            private Integer index;

        }
        H2QueryBuilderDDL h2QueryBuilderDDL = new H2QueryBuilderDDL();

        //when, then
        assertThat(h2QueryBuilderDDL.buildCreateQuery(Person.class)).isEqualTo(
                "CREATE TABLE users (id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY, nick_name VARCHAR(255), old INTEGER, email VARCHAR(255) NOT NULL);"
        );
    }

    @DisplayName("drop쿼리를 생성한다.")
    @Test
    void createDropQueryTest() {
        //given
        @Table(name = "users")
        @Entity
        class Person {

            @Id
            @GeneratedValue(strategy = GenerationType.IDENTITY)
            private Long id;

            @Column(name = "nick_name")
            private String name;

            @Column(name = "old")
            private Integer age;

            @Column(nullable = false)
            private String email;

            @Transient
            private Integer index;

        }
        H2QueryBuilderDDL h2QueryBuilderDDL = new H2QueryBuilderDDL();

        //when, then
        assertThat(h2QueryBuilderDDL.buildDropQuery(Person.class)).isEqualTo(
                "DROP TABLE users;"
        );
    }

    @DisplayName("drop쿼리를 생성할시 @Entity가 없다면 예외를 발생시킨다.")
    @Test
    void createDropQueryNotExistEntityThrowExceptionTest() {
        //given
        @Table(name = "users")
        class Person {

            @Id
            @GeneratedValue(strategy = GenerationType.IDENTITY)
            private Long id;

            @Column(name = "nick_name")
            private String name;

            @Column(name = "old")
            private Integer age;

            @Column(nullable = false)
            private String email;

            @Transient
            private Integer index;

        }
        H2QueryBuilderDDL h2QueryBuilderDDL = new H2QueryBuilderDDL();

        //when, then
        assertThatThrownBy(() -> h2QueryBuilderDDL.buildDropQuery(Person.class))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessage("@Entity 어노테이션이 존재하지 않습니다.");
    }

}
```