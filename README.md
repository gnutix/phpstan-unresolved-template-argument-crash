# PHPStan 2.3.x-dev: internal error with `unresolvedTemplateArguments`

Reported upstream as [phpstan/phpstan#15331](https://github.com/phpstan/phpstan/issues/15331).

With `featureToggles.unresolvedTemplateArguments` on (it is part of bleeding edge
on `2.3.x`), analysing [`src/repro.php`](src/repro.php) crashes PHPStan, even at
level 0:

```
Internal error: The initial type of an unresolved template argument is never itself unresolved.
```

Turning the toggle off makes the error go away. Found on `2.3.x-dev@7c94037`
while analysing a real codebase (a generic `Timeline<T>` class with a
`zip(self<U> ...$others): self<list<T|U|null>>` method).

## Reproduce

```bash
composer install
vendor/bin/phpstan analyse -v --debug
```

The [CI workflow](.github/workflows/phpstan.yml) runs the same thing.

## What it takes

All of these are needed; removing any one of them makes the crash disappear:

- a method template `U` inferred from **two** arguments (`Box::of(2)` and
  `Box::of(3)`; the same literal twice crashes too, a single argument doesn't);
- a return type that mixes it with the class template (`self<T|U>`; `self<U>` is fine);
- an **impure** method called on the result (`->get()`): `SimpleImpurePoint`
  asks for the class display name, which describes the union and generalizes
  the unresolved argument.

```php
/** @template-covariant T */
final class Box
{
    /** @template U @param U $value @return self<U> */
    public static function of(mixed $value): self { return new self(); }

    /** @template U @param self<U> $a @param self<U> $b @return self<T|U> */
    public function zip(self $a, self $b): self { return new self(); }

    public function get(): void {}
}

Box::of(1)->zip(Box::of(2), Box::of(3))->get(); // 💥
```

## Stack trace

```
PHPStan\ShouldNotHappenException: The initial type of an unresolved template argument is never itself unresolved.
  in src/Type/Generic/UnresolvedTemplateArgumentType.php:64
#0 src/Type/Generic/UnresolvedTemplateArgumentType.php(93): UnresolvedTemplateArgumentType->__construct()
#1 src/Type/Generic/UnresolvedTemplateArgumentType.php(224): UnresolvedTemplateArgumentType->withInitialType()
#2 src/Type/UnionType.php(462): UnresolvedTemplateArgumentType->generalize()
#3 [internal function]: UnionType::{closure:{closure:UnionType::describe():459}:460}()
#4 src/Type/UnionType.php(460): array_map()
#5 src/Type/VerbosityLevel.php(218): UnionType->{closure:UnionType::describe():459}()
#6 src/Type/UnionType.php(459): VerbosityLevel->handle()
#7 src/Type/Generic/TypeProjectionHelper.php(15): UnionType->describe()
#8 src/Reflection/ClassReflection.php(330): TypeProjectionHelper::describe()
#9 src/Reflection/Callables/SimpleImpurePoint.php(108): ClassReflection->getDisplayName()
#10 src/Analyser/ExprHandler/MethodCallHandler.php(194): SimpleImpurePoint::createFromVariant()
#11 src/Analyser/NodeScopeResolver.php(638): MethodCallHandler->processExpr()
...
#16 src/Analyser/StatementsHandler.php(398): StatementsHandler->processStatementStep()
```
