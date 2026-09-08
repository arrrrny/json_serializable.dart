The json_serializable ecosystem provides three specialized annotations for controlling how specific data structures are serialized to and from JSON. While `@JsonSerializable` handles class-level configuration and `@JsonKey` provides field-level control, these three annotations address distinct needs: enum-level configuration, individual enum value overrides, and embedding JSON literals as compile-time constants. Understanding their interplay is essential for maintaining precise control over your JSON serialization layer.

## JsonEnum: Class-Level Enum Configuration

The `@JsonEnum` annotation provides configuration for how enum types are serialized across an entire enum declaration. Unlike `@JsonKey` which applies to individual fields within a class, `@JsonEnum` targets enum types directly, establishing baseline behavior for all enum values unless explicitly overridden.

### Core Properties

The annotation accepts three parameters that control enum serialization behavior:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `alwaysCreate` | `bool` | `false` | Whether to generate the enum map even if the enum isn't used in a `@JsonSerializable` class |
| `fieldRename` | `FieldRename` | `FieldRename.none` | Naming strategy for converting enum entry names to JSON values |
| `valueField` | `String?` | `null` | Field within an "enhanced enum" to use as serialization value |

The `fieldRename` parameter accepts five strategies defined in the `FieldRename` enum:

- **`none`**: Use the enum entry name without modification
- **`kebab`**: Convert `camelCase` to `kebab-case` (e.g., `noGood` → `no-good`)
- **`snake`**: Convert `camelCase` to `snake_case` (e.g., `noGood` → `no_good`)
- **`pascal`**: Convert `camelCase` to `PascalCase` (e.g., `noGood` → `NoGood`)
- **`screamingSnake`**: Convert `camelCase` to `SCREAMING_SNAKE_CASE` (e.g., `noGood` → `NO_GOOD`)

### When to Use alwaysCreate

By default (`alwaysCreate: false`), the generator only creates enum maps for enums that are actually used as fields in `@JsonSerializable` classes. Setting `alwaysCreate: true` forces generation of the enum map even for standalone enums. This is useful when you need direct access to the enum map for custom serialization logic or validation.

### The valueField Mechanism

For enhanced enums (enums with constructors and fields), `valueField` specifies which instance field should be used for serialization instead of the enum entry name. This provides a powerful abstraction layer where enums can maintain internal complexity while exposing clean serialization values.

Sources: [json_annotation/lib/src/json_enum.dart](json_annotation/lib/src/json_enum.dart#L17-L46), [json_annotation/lib/src/json_serializable.dart](json_annotation/lib/src/json_serializable.dart#L9-L35)

## JsonValue: Individual Enum Value Overrides

While `@JsonEnum` sets enum-wide defaults, `@JsonValue` provides per-entry control, allowing you to specify explicit serialization values for individual enum entries. This annotation takes precedence over `@JsonEnum`'s `fieldRename` and `valueField` settings.

### Value Types

`@JsonValue` accepts values of type `String`, `int`, or `null`. The type must be consistent across all entries in an enum for proper serialization. You cannot mix strings and integers within the same enum.

### Annotation Priority Hierarchy

The precedence order for determining an enum's serialized value follows this hierarchy:

1. **`@JsonValue` on the specific entry** (highest priority)
2. **`valueField` from `@JsonEnum`** (if specified and the field exists)
3. **`fieldRename` from `@JsonEnum`** (applied to the entry name)
4. **Entry name directly** (lowest priority)

This hierarchy allows for precise control while maintaining sensible defaults.

### Practical Examples

Consider an enum where most entries follow kebab-case naming, but one entry requires special handling:

```dart
@JsonEnum(fieldRename: FieldRename.kebab)
enum ConnectionStatus {
  @JsonValue('OPEN')  // Override takes precedence over fieldRename
  established,
  pending,
  terminated,
  @JsonValue('CLOSED')
  closed,
}
```

Here, `established` serializes as `"OPEN"` (due to `@JsonValue`), `pending` as `"pending"` (kebab-case from `@JsonEnum`), and `closed` as `"CLOSED"`.

Sources: [json_annotation/lib/src/json_value.dart](json_annotation/lib/src/json_value.dart#L5-L14), [json_serializable/lib/src/enum_utils.dart](json_serializable/lib/src/enum_utils.dart#L92-L137)

## JsonLiteral: Embedding JSON as Compile-Time Constants

`@JsonLiteral` provides a mechanism to embed JSON file contents directly into your Dart code as compile-time constants. This eliminates runtime file I/O and parsing overhead while maintaining a single source of truth for static JSON data.

### Implementation Details

The annotation must be applied to a getter (typically a top-level getter) and generates a private field containing the parsed JSON content as a Dart literal. The generated field name follows the pattern `_$[getterName]JsonLiteral`.

### asConst Parameter

The `asConst` parameter (default: `false`) determines whether the generated field is declared as `const` or `final`:

- **`asConst: true`**: Generates `const` field – content must be a valid compile-time constant
- **`asConst: false`**: Generates `final` field – allows non-constant values

### Path Resolution

The `path` parameter specifies a relative path from the annotated Dart file to the source JSON file. The generator resolves this path relative to the source file's directory, allowing flexible organization of JSON assets.

### Usage Pattern

```dart
@JsonLiteral('data.json')
Map get glossaryData => _$glossaryDataJsonLiteral;

@JsonLiteral('data.json', asConst: true)
Map get asConstGlossaryData => _$asConstGlossaryDataJsonLiteral;
```

The generator reads the referenced JSON file during build time and produces corresponding Dart code:

```dart
final _$glossaryDataJsonLiteral = [/* parsed content */];
const _$asConstGlossaryDataJsonLiteral = [/* parsed content as const */];
```

Sources: [json_annotation/lib/src/json_literal.dart](json_annotation/lib/src/json_literal.dart#L12-L32), [json_serializable/lib/src/json_literal_generator.dart](json_serializable/lib/src/json_literal_generator.dart#L20-L49)

## Integration with Generated Code

The three annotations work together within the json_serializable code generation pipeline. Understanding how the generator processes these annotations helps in designing effective serialization strategies.

### Enum Processing Pipeline

The `JsonEnumGenerator` and `JsonSerializableGenerator` collaborate to process enum annotations:

1. **`JsonEnumGenerator`** processes `@JsonEnum` annotations, generating standalone enum maps when `alwaysCreate: true`
2. **`JsonSerializableGenerator`** generates serialization code for classes, referencing enum maps for enum fields
3. **`enumValueMapFromType`** in `enum_utils.dart` resolves the final serialized value for each enum entry based on the priority hierarchy

### Generated Code Structure

For an enum like:

```dart
@JsonEnum(alwaysCreate: true, fieldRename: FieldRename.kebab)
enum DayType { noGood, rotten, veryBad }
```

The generator produces:

```dart
const _$DayTypeEnumMap = {
  DayType.noGood: 'no-good',
  DayType.rotten: 'rotten',
  DayType.veryBad: 'very-bad',
};
```

This map is then used by `$enumDecode` and `$enumDecodeNullable` helpers during deserialization.

### Error Handling and Validation

The generator validates `valueField` references at build time, throwing descriptive errors for invalid configurations:

- **Invalid field name**: When `valueField` references a non-existent field
- **Unsupported type**: When `valueField` references a field with type other than `String`, `int`, or `null`
- **Static field**: When `valueField` references a static field instead of an instance field

Sources: [json_serializable/lib/src/json_enum_generator.dart](json_serializable/lib/src/json_enum_generator.dart#L20-L36), [json_serializable/lib/src/enum_utils.dart](json_serializable/lib/src/enum_utils.dart#L50-L88)

## Best Practices and Common Patterns

### Combining Annotations

For complex enums requiring both default naming and individual overrides:

```dart
@JsonEnum(alwaysCreate: true, fieldRename: FieldRename.snake)
enum HttpMethod {
  @JsonValue('GET')
  get,
  @JsonValue('POST')
  post,
  put,  // Serializes as 'put' (snake_case doesn't change single words)
  delete,
}
```

### Enhanced Enums with valueField

For enums that need to serialize based on a specific field:

```dart
@JsonEnum(alwaysCreate: true, valueField: 'statusCode')
enum HttpStatus {
  @JsonValue(404)
  notFound(404, 'Not Found'),
  @JsonValue(200)
  ok(200, 'OK');

  const HttpStatus(this.statusCode, this.message);
  final int statusCode;
  final String message;
}
```

### Standalone JSON Constants

For static configuration data or test fixtures:

```dart
@JsonLiteral('config/default_settings.json', asConst: true)
Map get defaultSettings => _$defaultSettingsJsonLiteral;

@JsonLiteral('test/fixtures/sample_response.json')
Map get sampleResponse => _$sampleResponseJsonLiteral;
```

Sources: [json_serializable/test/integration/json_enum_example.dart](json_serializable/test/integration/json_enum_example.dart#L10-L42), [json_serializable/test/literal/json_literal.dart](json_serializable/test/literal/json_literal.dart#L12-L18)

## Next Steps

Now that you understand the enum and literal annotation system, explore how these integrate with the broader serialization architecture:

- **[JsonConverter for Custom Type Serialization](7-jsonconverter-for-custom-type-serialization)**: Learn how to create custom converters for types not natively supported
- **[Enum Serialization and EnumHelper](15-enum-serialization-and-enumhelper)**: Deep dive into the internal enum handling mechanisms
- **[JsonKey Field Annotation Reference](5-jsonkey-field-annotation-reference)**: Understand how field-level annotations interact with enum serialization