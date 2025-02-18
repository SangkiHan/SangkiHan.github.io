---
title: QueryBuilder DML 구현
author: SangkiHan
date: 2024-10-15 11:20:00 +0900
categories: [NEXT-STEP, JPA]
tags: [NEXT-STEP]
---
------------

[Github Repo](https://github.com/SangkiHan/jpa-query-builder)

## DDL이란
DML은 Data Manipulation Language의 약자로, 데이터베이스에서 데이터를 조작하기 위한 SQL 명령어들을 포함하는 언어이다. DML은 데이터의 입력, 수정, 삭제 및 검색과 관련된 다양한 작업을 수행하는 데 사용된다.

+ SELECT: 데이터베이스에서 데이터를 조회할 때 사용합니다.
+ INSERT: 데이터베이스에 새로운 데이터를 추가할 때 사용합니다.
+ UPDATE: 기존 데이터의 값을 수정할 때 사용합니다.
+ DELETE: 데이터베이스에서 특정 데이터를 삭제할 때 사용합니다.

이번과정에는 Entity를 입력받아 SELECT, INSERT, DELETE 쿼리를 구현한다.

### DmlColumnData
DmlColumnData 객체를 생성하여 DML 쿼리에 필요한 데이터를 담게 하였다.
``` java
public class DMLColumnData {

    private final String columnName;
    private Class<?> columnType;
    private Object columnValue;
    private boolean isPrimaryKey;

    public DMLColumnData(String columnName, Class<?> columnType, Object columnValue, boolean isPrimaryKey) {

    this.columnName = columnName;
        this.columnType = columnType;
        this.columnValue = columnValue;
        this.isPrimaryKey = isPrimaryKey;
    }

    public DMLColumnData(String columnName, Class<?> columnType, boolean isPrimaryKey) {
        this.columnName = columnName;
        this.columnType = columnType;
        this.isPrimaryKey = isPrimaryKey;
    }

    public DMLColumnData(String columnName) {
        this.columnName = columnName;
    }

    public static DMLColumnData creatInstancePkColumn(String columnName, Class<?> columnType) {
        return new DMLColumnData(columnName, columnType, true);
    }

    public static DMLColumnData creatEntityPkColumn(String columnName, Class<?> columnType, Object columnValue) {
        return new DMLColumnData(columnName, columnType, columnValue, true);
    }

    public static DMLColumnData creatInstanceColumn(String columnName, Class<?> columnType, Object columnValue) {
        return new DMLColumnData(columnName, columnType, columnValue, false);
    }

    public static DMLColumnData createEntityColumn(String columnName) {
        return new DMLColumnData(columnName);
    }

    public String getColumnName() {
        return columnName;
    }

    public Class<?> getColumnType() {
        return columnType;
    }

    public Object getColumnValue() {
        return columnValue;
    }

    public boolean isPrimaryKey() {
        return isPrimaryKey;
    }
}
```

### QueryBuilder

이전 DML을 구현했을 때의 메소드가 DDL을 구현할때에도 공통적으로 사용이 되어 하나의 클래스에 공통 메소드를 묶어버렸다.

``` java
public class QueryBuilder {

    private final static String NOT_EXIST_ENTITY_ANNOTATION = "@Entity 어노테이션이 존재하지 않습니다.";

    //Entity 어노테이션 여부를 확인한다.
    protected void confirmEntityAnnotation(Class<?> entityClass) {
        if (!entityClass.isAnnotationPresent(Entity.class)) {
            throw new IllegalArgumentException(NOT_EXIST_ENTITY_ANNOTATION);
        }
    }

    //Table 어노테이션 여부를 확인한다.
    protected String getTableName(Class<?> entityClass) {
        if (entityClass.isAnnotationPresent(Table.class)) {
            Table table = entityClass.getAnnotation(Table.class);
            return table.name();
        }
        return entityClass.getSimpleName();
    }

}
```

### QueryBuilderDML

어떤 DB가 들어와도 insert, select, delete 쿼리 생성 기능은 필수적으로 존재 하기 때문에 interface를 생성하여 DB에 따라 유연하게 동작하도록 하였다.

``` java
public interface QueryBuilderDML {

    //Insert 쿼리를 생성한다.
    <T> String buildInsertQuery(T entityInstance);

    //findAll 쿼리를 생성한다.
    String buildFindAllQuery(Class<?> entityClass);

    //findId 쿼리를 생성한다.
    String buildFindByIdQuery(Class<?> entityClass, Object id);

    //delete 쿼리를 생성한다.
    String buildDeleteByIdQuery(Class<?> entityClass, Object id);

    String buildDeleteQuery(Class<?> entityClass);
}
```

``` java
public class H2QueryBuilderDML extends QueryBuilder implements QueryBuilderDML {

    private final static String INSERT_QUERY = "INSERT INTO {tableName} ({columnNames}) VALUES ({values});";
    private final static String FIND_ALL_QUERY = "SELECT {columnNames} FROM {tableName};";
    private final static String FIND_BY_ID_QUERY = "SELECT {columnNames} FROM {tableName} WHERE {entityPkName} = {values};";
    private final static String DELETE_BY_ID_QUERY = "DELETE FROM {tableName} WHERE {entityPkName} = {values};";
    private final static String DELETE_QUERY = "DELETE FROM {tableName};";
    private final static String COMMA = ", ";
    private final static String TABLE_NAME = "{tableName}";
    private final static String COLUMN_NAMES = "{columnNames}";
    private final static String VALUES = "{values}";
    private final static String ENTITY_PK_NAME = "{entityPkName}";
    private final static String SINGLE_QUOTE = "'";

    //insert 쿼리를 생성한다. Insert 쿼리는 인스턴스의 데이터를 받아야함
    @Override
    public <T> String buildInsertQuery(T entityInstance) {
        confirmEntityAnnotation(entityInstance.getClass());
        return insertQuery(getTableName(entityInstance.getClass()), getInstanceColumnData(entityInstance));
    }

    //findAll 쿼리를 생성한다.
    @Override
    public String buildFindAllQuery(Class<?> entityClass) {
        confirmEntityAnnotation(entityClass);
        return findAllQuery(getTableName(entityClass), getEntityColumnData(entityClass));
    }

    //findById 쿼리를 생성한다.
    @Override
    public String buildFindByIdQuery(Class<?> entityClass, Object id) {
        confirmEntityAnnotation(entityClass);
        return findByIdQuery(getTableName(entityClass), getEntityColumnData(entityClass), id);
    }

    @Override
    public String buildDeleteByIdQuery(Class<?> entityClass, Object id) {
        confirmEntityAnnotation(entityClass);
        return deleteByIdQuery(getTableName(entityClass), getEntityColumnData(entityClass), id);
    }

    @Override
    public String buildDeleteQuery(Class<?> entityClass) {
        confirmEntityAnnotation(entityClass);
        return deleteQuery(getTableName(entityClass));
    }

    //insert쿼리문을 생성한다.
    private String insertQuery(String tableName, List<DMLColumnData> columns) {
        //컬럼명을 가져온다.
        String columnNames = columns.stream()
                .map(DMLColumnData::getColumnName)
                .collect(Collectors.joining(COMMA));

        //Insert 할 Value 값들을 가져온다.
        String columnValues = columns.stream()
                .map(dmlColumnData -> {
                    Object value = dmlColumnData.getColumnValue();
                    if (dmlColumnData.getColumnType() == String.class) { //데이터 타입이 String 이면 작은 따옴표로 묶어준다.
                    return SINGLE_QUOTE + value + SINGLE_QUOTE;
                    }
                    return String.valueOf(value);
                })
                .collect(Collectors.joining(COMMA));

        return INSERT_QUERY.replace(TABLE_NAME, tableName)
                .replace(COLUMN_NAMES, columnNames)
                .replace(VALUES, columnValues);
    }

    //findAll 쿼리문을 생성한다.
    private String findAllQuery(String tableName, List<DMLColumnData> columns) {
        //컬럼명을 가져온다.
        String columnNames = columns.stream()
                .map(DMLColumnData::getColumnName)
                .collect(Collectors.joining(COMMA));

        return FIND_ALL_QUERY.replace(TABLE_NAME, tableName)
                .replace(COLUMN_NAMES, columnNames);
    }

    //findAll 쿼리문을 생성한다.
    private String findByIdQuery(String tableName, List<DMLColumnData> columns, Object id) {
        //컬럼명을 가져온다.
        String columnNames = columns.stream()
                .map(DMLColumnData::getColumnName)
                .collect(Collectors.joining(COMMA));

        // PK 컬럼명을 가져온다.
        String entityPkName = columns.stream()
                .filter(DMLColumnData::isPrimaryKey)
                .map(DMLColumnData::getColumnName)
                .findFirst()
                .orElseThrow(() -> new RuntimeException("PK 컬럼을 찾을 수 없습니다."));

        if (id instanceof String) //데이터 타입이 String 이면 작은 따옴표로 묶어준다.
        id = SINGLE_QUOTE + id + SINGLE_QUOTE;

        return FIND_BY_ID_QUERY.replace(TABLE_NAME, tableName)
                .replace(COLUMN_NAMES, columnNames)
                .replace(ENTITY_PK_NAME, entityPkName)
                .replace(VALUES, String.valueOf(id));
    }

    //delete 쿼리문을 생성한다.
    private String deleteByIdQuery(String tableName, List<DMLColumnData> columns, Object id) {
        // PK 컬럼명을 가져온다.
        String entityPkName = columns.stream()
                .filter(DMLColumnData::isPrimaryKey)
                .map(DMLColumnData::getColumnName)
                .findFirst()
                .orElseThrow(() -> new RuntimeException("PK 컬럼을 찾을 수 없습니다."));

        if (id instanceof String) //데이터 타입이 String 이면 작은 따옴표로 묶어준다.
            id = SINGLE_QUOTE + id + SINGLE_QUOTE;

        return DELETE_BY_ID_QUERY.replace(TABLE_NAME, tableName)
                .replace(ENTITY_PK_NAME, entityPkName)
                .replace(VALUES, String.valueOf(id));
    }

    //delete 쿼리문을 생성한다.
    private String deleteQuery(String tableName) {
        return DELETE_QUERY.replace(TABLE_NAME, tableName);
    }

    //Id 어노테이션을 primarykey로 가져온다.
    private void getEntityPrimaryKey(List<DMLColumnData> DMLColumnDataList, Field field) {
        if (field.isAnnotationPresent(Id.class)) {
            DMLColumnDataList.add(DMLColumnData.creatInstancePkColumn(field.getName(), field.getType()));
        }
    }

    //Id 어노테이션을 primarykey로 가져온다.
    private <T> void getInstancePrimaryKey(List<DMLColumnData> DMLColumnDataList, Field field, T entityInstance) {
        try {
            if (field.isAnnotationPresent(Id.class)) {
                field.setAccessible(true);
                DMLColumnDataList.add(DMLColumnData.creatEntityPkColumn(field.getName(), field.getType(), field.get(entityInstance)));
            }
        } catch (IllegalAccessException e) {
            throw new RuntimeException("필드 값을 가져오는 중 에러가 발생했습니다. : " + field.getName(), e);
        }
    }

    //Entity Class의 컬럼명과 컬럼데이터타입을 가져온다.
    private List<DMLColumnData> getEntityColumnData(Class<?> entityClass) {
        Field[] fields = entityClass.getDeclaredFields();
        List<DMLColumnData> DMLColumnDataList = new ArrayList<>();
        for (Field field : fields) {
            getEntityPrimaryKey(DMLColumnDataList, field);
            createDMLEntityColumnData(DMLColumnDataList, field);
        }
        return DMLColumnDataList;
    }

    //Entity 인스턴스의 컬럼명과 컬럼데이터타입, 컬럼데이터를 가져온다.
    private <T> List<DMLColumnData> getInstanceColumnData(T entityInstance) {
        Field[] fields = entityInstance.getClass().getDeclaredFields();
        List<DMLColumnData> DMLColumnDataList = new ArrayList<>();
        for (Field field : fields) {
            getInstancePrimaryKey(DMLColumnDataList, field, entityInstance);
            createDMLInstanceColumnData(DMLColumnDataList, field, entityInstance);
        }
        return DMLColumnDataList;
    }

    //Entity 내부 필드를 확인하여 필드명을 가져온다.
    private void createDMLEntityColumnData(List<DMLColumnData> DMLColumnDataList, Field field) {
        if (field.isAnnotationPresent(Transient.class) || field.isAnnotationPresent(Id.class))
            return; // @Transient인 경우 검증하지 않음

        String columnName = field.getName();

        if (field.isAnnotationPresent(Column.class)) {
            Column column = field.getAnnotation(Column.class);
            columnName = column.name().isEmpty() ? columnName : column.name();
        }

        DMLColumnDataList.add(DMLColumnData.createEntityColumn(columnName));
    }

    //인스턴스 내부 데이터를 확인하여 컬럼 데이터를 가져온다.
    private <T> void createDMLInstanceColumnData(List<DMLColumnData> DMLColumnDataList, Field field, T entityInstance) {
        try {
            if (field.isAnnotationPresent(Transient.class) || field.isAnnotationPresent(Id.class))
              return; // @Transient인 경우 검증하지 않음

            String columnName = field.getName();

            if (field.isAnnotationPresent(Column.class)) {
                Column column = field.getAnnotation(Column.class);
                columnName = column.name().isEmpty() ? columnName : column.name();
            }

            field.setAccessible(true);
            DMLColumnDataList.add(DMLColumnData.creatInstanceColumn(columnName, field.getType(), field.get(entityInstance)));
        } catch (IllegalAccessException e) {
            throw new RuntimeException("필드 값을 가져오는 중 에러가 발생했습니다. : " + field.getName(), e);
        }
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

#### H2QueryBuilderDML 테스트


``` java
public class H2QueryBuilderDMLTest {
    @DisplayName("Insert 쿼리 문자열 생성하기")
    @Test
    void buildInsertTest() {
        //given
        QueryBuilderDML queryBuilderDML = new H2QueryBuilderDML();

        Person person = new Person(1L, "sangki", 29, "test@test.com", 1);

        //when, then
        assertThat(queryBuilderDML.buildInsertQuery(person))
                .isEqualTo("INSERT INTO users (id, nick_name, old, email) VALUES (1, 'sangki', 29, 'test@test.com');");
    }

    @DisplayName("findAll 쿼리 문자열 생성하기")
    @Test
    void buildFindAllTest() {
        //given
        QueryBuilderDML queryBuilderDML = new H2QueryBuilderDML();

        //when, then
        assertThat(queryBuilderDML.buildFindAllQuery(Person.class))
                .isEqualTo("SELECT id, nick_name, old, email FROM users;");
    }

    @DisplayName("findById 쿼리 문자열 생성하기")
    @Test
    void buildFindByIdTest() {
        //given
        QueryBuilderDML queryBuilderDML = new H2QueryBuilderDML();

        //when, then
        assertThat(queryBuilderDML.buildFindByIdQuery(Person.class, 1))
                .isEqualTo("SELECT id, nick_name, old, email FROM users WHERE id = 1;");
    }

    @DisplayName("findById 쿼리 문자열 생성할시 id가 String이면 작은따옴표로 묶어준다.")
    @Test
    void buildFindByIdStringTest() {
        //given
        QueryBuilderDML queryBuilderDML = new H2QueryBuilderDML();

        //when, then
        assertThat(queryBuilderDML.buildFindByIdQuery(Person.class, "sangki"))
                .isEqualTo("SELECT id, nick_name, old, email FROM users WHERE id = 'sangki';");
    }

    @DisplayName("deleteById 쿼리 문자열 생성한다.")
    @Test
    void buildDeleteByIdTest() {
        //given
        QueryBuilderDML queryBuilderDML = new H2QueryBuilderDML();

        //when, then
        assertThat(queryBuilderDML.buildDeleteByIdQuery(Person.class, "sangki"))
                .isEqualTo("DELETE FROM users WHERE id = 'sangki';");
    }

    @DisplayName("deleteAll 쿼리 문자열 생성한다.")
    @Test
    void buildDeleteTest() {
        //given
        QueryBuilderDML queryBuilderDML = new H2QueryBuilderDML();

        //when, then
        assertThat(queryBuilderDML.buildDeleteQuery(Person.class))
                .isEqualTo("DELETE FROM users;");
    }
}
```