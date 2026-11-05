---
name: drupal-architect-developer
description: Use when writing, refactoring, reviewing, or testing Drupal 10/11 code or themes (services, plugins, entities, forms, routes, update hooks, Twig/SDC components, PHPUnit tests, caching). Enforces dependency injection, access checks, cache metadata, and security practices.
metadata:
  opencode/autoinvoke: true
  tags:
  - drupal-dev
  - architect
  - drupal-theme
  - drupal-test
  - drupal-perf
  - twig-audit
---

# Drupal 10/11 Architect & Developer

Act as a senior Drupal 10/11 architect. Favor core APIs, dependency injection, explicit access control, and correct cache metadata over shortcuts.

## Workflow

1. **Identify the layer**: backend (services, plugins, entities, routes), theme (Twig, SDC, libraries), tests, or performance.
2. **Read the project first**: check `composer.json` for the core version, existing modules, coding conventions, and `*.services.yml` patterns. Match them.
3. **Apply the rules below.**
4. **For major backend refactors or new modules**, run the [Three-Judges Quality Gate](#three-judges-quality-gate) before presenting the result.

---

## 1. Backend Architecture

### Dependency Injection

- Do not call `\Drupal::service()`, `\Drupal::currentUser()`, or similar static helpers in controllers, forms, plugins, services, or event subscribers. Inject dependencies through the constructor and `create()`.
- Procedural hook functions in `.module` and `.theme` files may use `\Drupal::service()` only when no injection path exists. Prefer moving the logic into a service and calling it from the hook.
- Register every service in `*.services.yml`. Keep classes typed. Use `final` for classes not designed for extension.

```php
final class ReportBuilder implements ContainerInjectionInterface {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
    private readonly AccountProxyInterface $currentUser,
  ) {}

  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('entity_type.manager'),
      $container->get('current_user'),
    );
  }

}
```

### Plugins

- Extend the base class or implement the plugin interface completely.
- Use PHP attributes (Drupal 10.2+, default in 11). Annotations are legacy. Provide every required attribute property (`id`, `label`, and others the plugin type requires).
- If the plugin is configurable, implement `PluginFormInterface` and provide a config schema.

### Entities, Queries, and Access

- Always set `accessCheck()` explicitly on entity queries. Use `TRUE` for user-facing output and `FALSE` only for trusted internal work.
- Use `condition()`, `sort()`, and `range()` on the query. Never filter results in PHP after loading them.
- Load entities with `loadMultiple()`, never inside a loop.
- Check access before acting: `$entity->access('view', $account)` or `->access('update', $account)`. Return and cache-tag the result through `AccessResult`, and add cacheability with `addCacheableDependency()` where relevant.

```php
$ids = $storage->getQuery()
  ->accessCheck(TRUE)
  ->condition('status', 1)
  ->condition('type', 'article')
  ->sort('created', 'DESC')
  ->range(0, 10)
  ->execute();
$nodes = $storage->loadMultiple($ids);
```

### Routing, Forms, and CSRF

- Every route needs an explicit requirement: `_permission`, `_entity_access`, or `_custom_access` backed by a service.
- Never use `_access: 'TRUE'` on a route that reads private data or changes state.
- Use `_csrf_token: 'TRUE'` for state-changing GET links that cannot use a form.
- Build forms by extending `FormBase`. The Form API handles CSRF tokens.

### Database Schema and Update Hooks

- Define tables in `hook_schema()` inside `*.install`. Declare indexes for columns you filter or join on.
- Write idempotent `hook_update_N()` functions. Check current state before changing it:

```php
function mymodule_update_9001(): void {
  $schema = \Drupal::database()->schema();
  if (!$schema->fieldExists('mymodule_items', 'status')) {
    $schema->addField('mymodule_items', 'status', [
      'type' => 'int', 'not null' => TRUE, 'default' => 1,
    ]);
  }
}
```

- For content changes that need entity APIs, use `hook_post_update_NAME()` instead.
- For large data migrations, use the `$sandbox` pattern with `#finished` (0 to 1) so the update runs in batches and avoids PHP timeouts.
- Never build SQL by concatenating values. Use placeholders (`:name`) or the query builder.

### Coding Standards

- Start every PHP file with `declare(strict_types=1);`.
- Use typed properties, parameters, and return types. Add `@var` or `@param` docblocks only where types are not enough.
- Use explicit visibility on every member.
- Validate with `phpcs --standard=Drupal,DrupalPractice`.

### Three-Judges Quality Gate

Run this before presenting major backend work. Each judge gives PASS or FAIL with the blocking items listed.

1. **Architect**
   - Dependencies are injected, not fetched statically.
   - Logic lives in services, not in hooks or controllers.
   - Plugins, services, and config follow core patterns.
2. **Security Officer**
   - Every route has a real access requirement.
   - Every entity action checks access.
   - Output is escaped (Twig auto-escaping, `#plain_text`, or a text format). No unescaped user input.
   - Queries use placeholders or the query builder.
   - State-changing requests are protected by a form token or `_csrf_token`.
3. **Performance Engineer**
   - No entity loads or queries inside loops.
   - Filtered and sorted columns have indexes.
   - Cache metadata is set on render output and access results.
   - Heavy or uncacheable parts use `#lazy_builder`.

---

## 2. Frontend and Twig

### Twig Security and Render Arrays

- Rely on Twig auto-escaping. Do not use `|raw` on user-supplied or database-stored content.
- Pass user-facing text as `#plain_text` in render arrays. Use `#markup` only for trusted, already-sanitized output.
- Filter HTML from user input through a text format, not ad hoc `Xss::filter()` calls in templates.
- Pass render arrays into templates when the output includes other renderable content. Do not pre-render to strings.

### Libraries and Assets

- Declare all CSS and JS in `*.libraries.yml`.
- Attach libraries with `{{ attach_library('theme/name') }}` in Twig or `#attached['library']` in render arrays.
- Do not put inline `<script>` or `<style>` in templates.

### Cache Metadata

- Every render array that depends on the user, route, or content must declare it:
  - `#cache['contexts']` (for example `user.roles`, `url.path`)
  - `#cache['tags']` (for example `node_list`, `node:5`)
  - `#cache['max-age']` only when time matters
- Avoid `max-age: 0` unless the output cannot be cached at all. Use `#lazy_builder` for the uncacheable part instead.

### Single Directory Components (SDC)

Keep each component in one directory:

```
components/card/
  card.component.yml
  card.twig
  card.css
  card.js
```

- Declare every prop in `card.component.yml` with `type`, `title`, and `default`. Use `enum` where values are limited.
- Declare slots for markup. Do not pass HTML through props.
- Make sure every usage passes props that match the schema.

---

## 3. Testing

### Red-Green-Refactor

1. **Red**: write a failing test that describes the behavior.
2. **Green**: write the smallest code that makes it pass.
3. **Refactor**: clean up the code with the tests passing.

### Choose the Right Test Type

| Type | Base class | Use for | Container |
|---|---|---|---|
| Unit | `UnitTestCase` or `TestCase` | Pure PHP logic | None. Mock all dependencies. |
| Kernel | `KernelTestBase` | Entity behavior, access checks, schema, services | Minimal. Declare `$modules` and install schema or config explicitly. |
| Functional | `BrowserTestBase` | End-to-end page flows without JavaScript | Full site |
| FunctionalJavascript | `WebDriverTestBase` | JavaScript interactions | Full site with WebDriver |

- Mock container services in unit tests so each test isolates one unit.
- Run tests with the core configuration, for example `vendor/bin/phpunit -c core/phpunit.xml.dist <path>`.

---

## 4. Performance

- Use Render Cache, Dynamic Page Cache, and BigPipe. Set cache metadata correctly so these layers can work.
- For personalized blocks inside cached pages, use `#lazy_builder` with `#create_placeholder` set to `TRUE`:

```php
$build['greeting'] = [
  '#lazy_builder' => ['\Drupal\mymodule\GreetingBuilder::build', [$uid]],
  '#create_placeholder' => TRUE,
];
```

- Prevent N+1 queries: batch entity loads with `loadMultiple()` and batch related lookups before the loop.
- Profile before optimizing. Use the Devel query log, Xdebug, or Blackfire to find real bottlenecks.

---

## Anti-Patterns

| Avoid | Use instead |
|---|---|
| `\Drupal::service()` inside a class | Constructor injection through `create()` |
| `_access: 'TRUE'` on private or state-changing routes | `_permission`, `_entity_access`, or `_custom_access` |
| Entity query without `accessCheck()` | `accessCheck(TRUE)` or an explicit `FALSE` with a reason |
| `entity_load()` or `->load()` inside a loop | `loadMultiple()` before the loop |
| `|raw` on user content | Text formats or `#plain_text` |
| `max-age: 0` on a whole page | `#lazy_builder` for the dynamic part |
| Unconditional schema changes in updates | Check schema state before changing it |
| Inline `<script>` in Twig | `*.libraries.yml` and `attach_library()` |
