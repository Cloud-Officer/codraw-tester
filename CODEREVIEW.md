# Code Review: codraw/tester

**Package:** `codraw/tester` (namespace `Draw\Component\Tester`)
**Reviewed:** all PHP source, `composer.json`, DI integration, PHPUnit/PHPStan extensions. `Tests/` skimmed for coverage assessment.

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
- **M7 (partial)** — `composer.json`: added `"php": ">=8.5"` to `require`; added `suggest` entries for `doctrine/orm` (DoctrineOrmTrait), `nesbot/carbon` (CarbonResetExtension) and `symfony/console` (commands / CommandTestTrait). Open items (deliberately not changed): `guzzlehttp/psr7` and `psr/http-message` are unused but were left in `require` since consumers may rely on them transitively; Symfony 7 support was not added (`^6.4.0` is the repo-wide convention).
- **M6** — `Command/TestsCoverageCheckCommand.php`: missing input file now reports the path the user actually typed (raw argument validated together with the `realpath()` result instead of after); a clover report with zero statements now throws a clear `RuntimeException` instead of an uncaught `DivisionByZeroError`; the below-threshold message no longer truncates fractional thresholds (`%d` → `%s`).
- **L1** — `Application/CommandTestTrait.php::execute()`: `COLUMNS` restoration wrapped in `try/finally` so it survives command/assertion failures, and a previously-unset `COLUMNS` is now unset again (`putenv('COLUMNS')`) instead of being set to an empty string.
- **L3** — `Data/ViolationListTester.php`: `code()` and `invalidValue()` now throw a `\LogicException` ("call addViolation() first") instead of silently writing a phantom violation at index `-1`.
- **L4** — `Command/GenerateAssertsDocumentationPageCommand.php` + `docs/asserts.rst`: extraction markers changed to `// example-start:` / `// example-end:` (with space) so all 94 `literalinclude` directives match the committed `AssertTrait.php`.
- **M2** — PHPUnit 12 compatibility of `AssertTrait`. Verified every wrapped method against the actual PHPUnit 12.0.0 and 12.4.0 `Assert` sources: only `assertStringNotMatchesFormat()` is removed in PHPUnit 12 (`assertStringNotMatchesFormatFile()` was also removed but is not wrapped — it is `"ignore": true`). The review's claim about `assertContainsOnly()`/`assertNotContainsOnly()` was incorrect: both still exist in PHPUnit 12.0–12.4 with the unchanged `(string $type, iterable $haystack, ?bool $isNativeType = null, string $message = '')` signature. Fix applied: `AssertTrait::assertStringNotMatchesFormat()` now guards with `method_exists(Assert::class, ...)` and calls `Assert::fail()` with an explicit "removed in PHPUnit 12" message instead of fataling with an opaque "call to undefined method"; the wrapper is kept (no BC break for PHPUnit 11 consumers) and the phpunit constraint untouched. `Command/GenerateTraitCommand.php` now emits the same guard for any method flagged with a `"removedIn"` key in `Resources/config/assert_methods.json` (key added for `assertStringNotMatchesFormat`), so regeneration preserves the guard. Open item: the generator reflects the *installed* `Assert` class, so regeneration must still run under PHPUnit 11 (under 12 the removed method cannot be reflected and `getMethod()` would throw). Verified with `php -l`; test suite not run (no `vendor/` present).
- **L6 (partial)** — `DoctrineOrmTrait.php`: a missing/empty `$dsn` with unset `DATABASE_URL` now throws a clear `RuntimeException` instead of feeding `false` into `DsnParser::parse()`. The misleading `?EntityManagerInterface` return type and the `orm:schema-tool:update --force` guard were left untouched (API/behavior changes).

Not fixed (out of scope for low-risk pass): M1 (callable discrimination change would alter behavior for consumers passing string callables), M3 (test-architecture redesign), M4/M5 (behavior changes in the autowire extension), L2 (removing trait lifecycle method could break consumers), L5 (stricter validation), L7 (documentation-only advisory on internal design), L8 (stricter validation).

### Validation pass (2026-07-20)

All fixes above were validated against a full local CI run; no fix required repair and no test fallout occurred:

- `composer install` resolves cleanly with the added `"php": "^8.5"` constraint (PHPUnit 12.5.31, Symfony 6.4 installed).
- `vendor/bin/phpunit` (with `DATABASE_URL` pointing at a local MySQL `codraw_tester` database): **OK, 40 tests, 105 assertions** — the full `Tests/` suite, including `TestsCoverageCheckCommandTest` (covers M6) and `AgainstJsonFileTesterTest`.
- PHPStan (level per `phpstan.dist.neon`): 5 errors, all verified pre-existing via `git stash` (identical without the fixes — `function.impossibleType` in `Application/CommandTestTrait.php:38`, `trait.unused` for `DoctrineOrmTrait`/`MockTrait`, `notIdentical.alwaysTrue` in `Tests/DataTesterTest.php:81`, `staticMethod.alreadyNarrowedType` in `Tests/ExampleTest.php:120`). The M2 guard actually *removed* one pre-existing error (`staticMethod.notFound` on `Assert::assertStringNotMatchesFormat()` under PHPUnit 12).
- `markdownlint-cli2`: 0 errors across all 6 Markdown files.
- The M1 quick fix was considered again and deliberately *not* applied: the changed discrimination path (string/array callables no longer invoked) is not pinned by any existing test.

## Overall Assessment

This is a small (~3.5k LOC), focused testing utility library: a fluent `DataTester` assertion wrapper built on `Draw\Component\Core\DataAccessor`, a generated `AssertTrait` mirroring PHPUnit's `Assert` API, console-command test scaffolding, JSON-snapshot and violation-list testers, PHPUnit 11 event-system extensions (attribute-based property autowiring, Carbon reset), a PHPStan extension, and a coverage-threshold CLI command. The code is clean, modern (PHP 8 syntax, PHPUnit 11 event API, attributes), and mostly well designed. No exploitable security issues were found — the package is test tooling, its commands operate on local files only, and XML parsing relies on libxml's safe-by-default entity handling (PHP >= 8.0). The notable problems are compatibility (wrappers for assertions removed in PHPUnit 12 while `^12.0` is allowed), dependency hygiene in `composer.json`, order-dependent static state in `ExtensionTestCase`, a callable/value ambiguity in `AgainstJsonFileTester`, and a fragile `debug_backtrace()` hack in the autowire extension. Grade: **B**.

---

## Findings

### Medium

#### M1. `AgainstJsonFileTester` executes expected string values that happen to be callables

`Data/AgainstJsonFileTester.php:40`

```php
if (!\is_callable($callable)) {
    $value = $callable;
    ...
}
```

The `$propertyPathsCheck` map accepts a `Constraint`, a callable, or a plain expected value. The discrimination uses `is_callable()`, so a plain *string* expected value that collides with a function name (`'date'`, `'trim'`, `'max'`, `'key'`, `'Foo::bar'`, or any invokable class name) is invoked as a callback with a `DataTester` argument instead of being compared, typically producing a confusing `TypeError` (or, worse, silently passing if the function tolerates the argument). Discriminate explicitly: only treat `\Closure`/objects with `__invoke` as callables, or require `Constraint`/`\Closure` for behavioral checks.

#### **[FIXED]** M2. `AssertTrait` wraps assertions removed/changed in PHPUnit 12, but composer allows `^12.0`

`AssertTrait.php:45` (`assertContainsOnly` with `?bool $isNativeType`), `AssertTrait.php:69` (`assertNotContainsOnly`), `AssertTrait.php:403` (`assertStringNotMatchesFormat`); `composer.json:22`

`composer.json` permits `phpunit/phpunit: ^11.3 || ^12.0`, but the generated trait still delegates to `Assert::assertStringNotMatchesFormat()` (deprecated in 11.x, removed in PHPUnit 12) and to `assertContainsOnly()`/`assertNotContainsOnly()` with the `$isNativeType` signature that was deprecated in 11.5 and removed/changed in 12 (replaced by `assertContainsOnly*` type-specific methods / `NativeType` enum). Under PHPUnit 12 these wrappers fatal with `Error: Call to undefined method` (or argument errors) at call time. The trait should be regenerated against the supported PHPUnit majors (the `Resources/config/assert_methods.json` + `draw:tester:dump-assert-methods` pipeline exists precisely for this), or the constraint narrowed to `^11.3` until it is.

#### M3. `ExtensionTestCase` relies on static state and test execution order

`Test/DependencyInjection/ExtensionTestCase.php:12-17, 43-48, 86-124`

`testDefinitionsMatchChecks()`/`testAliasesMatchChecks()` compare the container's definitions against `self::$definitions`/`self::$aliases`, which are only populated as a *side effect* of `testServiceDefinition()` data-provider cases running earlier in the same class, after being reset in `setUpBeforeClass()`. Under `--order-by=random`, `--process-isolation`, running a single filtered test, or parallel runners, the "match" tests run with empty/partial expected arrays and fail (or trivially pass if no definitions exist). The same pattern applies to the shared `self::$containerBuilder` built once from the first test's `getConfiguration()`. This works only under default sequential ordering; consider building expectations from `provideServiceDefinitionCases()` directly inside the match tests instead of accumulating static state.

#### M4. `SetUpAutowireExtension` locates the `TestCase` instance via `debug_backtrace()`

`PHPUnit/Extension/SetUpAutowire/SetUpAutowireExtension.php:42-51`

The PHPUnit event API deliberately does not expose the `TestCase` object, so the subscriber walks `debug_backtrace()` looking for a frame whose `object` is a `TestCase`. This couples the extension to PHPUnit's internal call stack; a minor internal refactoring in a PHPUnit patch release changes the stack shape and the code then **silently returns** (line 49-51) without autowiring — typed test properties stay uninitialized and tests fail with unrelated-looking `Error: Typed property ... must not be accessed before initialization`, or `postAutowire()` silently never runs. At minimum, fail loudly when a class implements `AutowiredInterface` but no instance can be found in the backtrace.

#### M5. Autowire attribute discovery misses private properties of parent classes and reuses attribute instances across tests

`PHPUnit/Extension/SetUpAutowire/SetUpAutowireExtension.php:75-94`

- `(new \ReflectionObject($testCase))->getProperties()` does not return `private` properties declared in *parent* classes, so `#[AutowireMock]`/`#[AutowireMockProperty]` on a private property of an abstract base test case is silently ignored (works if `protected`). A walk up `getParentClass()` is needed.
- The `$propertyAttributes` cache stores the instantiated attribute objects per class and reuses the *same instances* for every test of that class; any autowire attribute that keeps internal state would leak it between tests.
- Minor: `usort()` ascending + `array_reverse()` (lines 90-93) gives descending priority but also reverses the (stable) declaration order of equal-priority attributes; sorting with `$b::getPriority() <=> $a::getPriority()` would preserve declaration order.

#### **[FIXED]** M6. `TestsCoverageCheckCommand`: division by zero and degraded error message

`Command/TestsCoverageCheckCommand.php:26-29, 49`

- Line 49: `($checkedElements / $totalElements) * 100` throws an uncaught `DivisionByZeroError` when the clover report contains no `<file>/<metrics>` statements (empty project, wrong file, or a clover file whose structure doesn't match the XPath) — instead of a clear diagnostic.
- Line 26-29: `realpath()` returns `false` for a missing file, so the exception message becomes `Invalid input file provided ""`, losing the path the user actually typed. Validate the raw argument first, then `realpath()`.
- Cosmetic: line 52-55 formats the threshold with `%d`, truncating fractional thresholds (e.g. `85.5` prints as `85`).

#### **[PARTIALLY FIXED]** M7. `composer.json` dependency hygiene

`composer.json:18-25`

- `guzzlehttp/psr7` and `psr/http-message` are required but **nothing in the package references PSR-7 or Guzzle** (verified by grep) — dead weight forced onto every consumer.
- No `"php"` constraint is declared; the effective floor is only implied through `phpunit/phpunit`.
- Shipped classes hard-depend on packages that are only in `require-dev`: `Command/*` and `Application/CommandTestTrait` need `symfony/console`, `DoctrineOrmTrait` needs `doctrine/orm` + `doctrine/dbal`, `Test/DependencyInjection/*` need `symfony/dependency-injection`/`symfony/config`, and `CarbonResetExtension` needs `nesbot/carbon`. That's a legitimate optional-feature pattern, but `suggest` (lines 33-36) only mentions `symfony/cache` and `symfony/dependency-injection`; `symfony/console`, `doctrine/orm`, and `nesbot/carbon` should be suggested so consumers get a hint instead of a class-not-found.
- Only `symfony/*: ^6.4.0` is allowed — no Symfony 7 support, which blocks use of this testing library in Symfony 7 applications.

### Low

#### **[FIXED]** L1. `CommandTestTrait::execute()` COLUMNS handling leaks environment state

`Application/CommandTestTrait.php:250-254`

If the command (or an assertion inside `CommandTester::execute()`) throws, `putenv('COLUMNS='.$columns)` is skipped — no `try/finally` — leaving `COLUMNS=120` for subsequent tests. Also, when `COLUMNS` was not set, `getenv()` returns `false` and the restore executes `putenv('COLUMNS=')`, which *sets* it to an empty string rather than unsetting it (`putenv('COLUMNS')`).

#### L2. Dead/hazardous trait members in `CommandTestTrait`

`Application/CommandTestTrait.php:15, 59-62`

`private static int $argumentsCount` is written in `setUpBeforeClass()` and never read — dead code. More importantly, the trait defines `public static function setUpBeforeClass()`, which silently prevents a using test class from having its own `setUpBeforeClass()` without method aliasing; keeping lifecycle methods out of the trait (or documenting the conflict) would be safer.

#### **[FIXED]** L3. `ViolationListTester::code()`/`invalidValue()` corrupt state when called before `addViolation()`

`Data/ViolationListTester.php:37, 47`

`$this->violations[\count($this->violations) - 1]` with an empty list writes to index `-1`, creating a phantom violation entry (containing only `code`/`invalidValue`) that later drives an off-by-one count assertion and a nonsense property path `[-1].code`. Guard with an exception ("call addViolation() first").

#### **[FIXED]** L4. Documentation extraction markers don't match the committed `AssertTrait.php`

`Command/GenerateAssertsDocumentationPageCommand.php:57-58` vs `AssertTrait.php:17` (and `docs/asserts.rst:20`)

The generated RST uses `:start-after: //example-start: assertX` (no space), but the committed trait contains `// example-start: assertX` (space after `//`, likely re-formatted by php-cs-fixer after generation). Sphinx `literalinclude` matches substrings per line, so none of the 94 markers in `docs/asserts.rst` match the source file and the docs build cannot extract the examples.

#### L5. `AutowireMock` doesn't verify the intersection actually contains `MockObject`

`PHPUnit/Extension/SetUpAutowire/AutowireMock.php:32-47`

For a property typed `Foo&Bar` (no `MockObject` member), the loop silently mocks the first type and stops, instead of rejecting the type hint as the error messages promise. Also, if the loop somehow completes without acting, the method returns leaving the property uninitialized with no error.

#### **[PARTIALLY FIXED]** L6. `DoctrineOrmTrait` rough edges

`DoctrineOrmTrait.php:24-53`

- Return type is `?EntityManagerInterface` but the method can never return `null` — misleading API forcing needless null-checks.
- `getenv('DATABASE_URL')` returning `false` flows into `DsnParser::parse()` producing a confusing `TypeError` rather than a "DATABASE_URL not set" message.
- Running `orm:schema-tool:update --force` against whatever `DATABASE_URL` points to is a footgun if the env var targets a non-test database; a guard (e.g. require `_test` suffix or explicit opt-in) would be prudent for a shared trait.

#### L7. `MockTrait::withConsecutive()` fragility

`MockTrait.php:35-77`

The shared-counter scheme (`$callbackCall`, `$mockedMethodCall` captured by reference across all yielded `Callback` constraints) assumes each constraint is evaluated exactly once per mocked invocation and in argument order; any constraint re-evaluation desynchronizes the counters and validates arguments against the wrong call. Additionally, arguments a consecutive call has *beyond* the first call's argument count are never validated (only keys of `$firstCallArguments` are iterated), despite the assertion at lines 38-43 implying they're supported. Fine as a pragmatic PHPUnit-10+ shim, but worth documenting the constraints.

#### L8. `DataTester::each()` silently accepts non-iterable data

`DataTester.php:44-51`

`foreach` over a scalar emits only a warning and iterates zero times, so `each()` on unexpected scalar data "passes" without executing any assertion. Asserting `is_iterable()` first would fail fast.

---

## Strengths

- **Clean fluent API design**: `DataTester` is immutable per navigation step (`path()`/`transform()` return new instances), composes well (`test()`, `each()`, `ifPathIsReadable()`), and produces helpful failure messages including the JSON-encoded data (`DataTester.php:53-75`).
- **Code-generation pipeline** (`DumpAssertMethodsCommand` + `GenerateTraitCommand` + `Resources/config/assert_methods.json`) keeps the ~90-method `AssertTrait` mechanically in sync with PHPUnit's `Assert`, with per-method opt-out and deprecation detection — much better than hand-maintaining wrappers.
- **Modern PHPUnit integration**: extensions use the PHPUnit 11 event system (`TestPreparedSubscriber`, `FinishedSubscriber`) rather than the removed hook interfaces; the attribute-based autowiring design (`AutowireInterface` + priorities + `AutowireConfigurableInterface` + `AutowiredCompletionAwareInterface`) is genuinely extensible.
- **Tooling coherence**: ships a PHPStan `ReadWritePropertiesExtension` (`PHPStan/Rules/Properties/AutowireReadWritePropertiesExtension.php`) so autowired test properties don't trigger uninitialized/never-written false positives, plus a `.phpstorm.meta.php` for `mockProperty()` return-type inference.
- **Docs as executable code**: `Tests/ExampleTest.php` doubles as the source for documentation snippets via start/end markers, so examples are continuously executed.
- `ext-simplexml` is correctly declared for the coverage command; PSR-4 autoload from package root is consistent; `withConsecutive()` restores a widely-missed PHPUnit capability.

---

## Test Coverage

**Well covered:**

- `DataTester` / inherited `DataAccessor` behavior: `Tests/DataTesterTest.php` and `Tests/ExampleTest.php` cover `path()`, chaining, `each()`, `transform()`, `ifPathIsReadable()`, readability asserts, `assertThat()`, and callable-object testers.
- `TestsCoverageCheckCommand`: `Tests/Command/TestsCoverageCheckCommandTest.php` covers argument definition, missing file, invalid percentage, below-threshold (exit 1) and above-threshold (exit 0) paths against a real clover fixture — and dogfoods `CommandTestTrait` in the process.
- `AgainstJsonFileTester`: `Tests/Data/AgainstJsonFileTesterTest.php` covers exact match, mismatch, property-path checks via value, constraint and callable, and failure messages.
- `TesterIntegration`: covered via `IntegrationTestCase` from `codraw/dependency-injection`.

**Not covered (no direct tests):**

- `MockTrait` — neither `withConsecutive()` (the most intricate logic in the package) nor `mockProperty()`.
- The PHPUnit extensions: `SetUpAutowireExtension`, `AutowireMock`, `AutowireMockProperty`, `CarbonResetExtension` (understandably hard to test, but they carry real logic — priority sorting, backtrace discovery, intersection-type parsing).
- `DoctrineOrmTrait`, `ViolationListTester`, `CommandDataTester` (only exercised indirectly, if at all).
- The generator commands (`GenerateTraitCommand`, `DumpAssertMethodsCommand`, `GenerateAssertsDocumentationPageCommand`).
- The reusable base classes `ExtensionTestCase` / `ConfigurationTestCase`, and the PHPStan extension.
- `AssertTrait`'s generated delegate methods are untested individually — acceptable given they are generated one-line delegates, but a smoke test iterating the methods would catch PHPUnit signature drift like finding M2 automatically.
