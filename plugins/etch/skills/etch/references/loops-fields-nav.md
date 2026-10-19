# etch.loops, etch.fields, etch.navigation, etch.ui, etch.history

Sources: https://docs.etchwp.com/public-api/loops ,
https://docs.etchwp.com/public-api/fields ,
https://docs.etchwp.com/public-api/navigation ,
https://docs.etchwp.com/public-api/ui-and-history

## etch.loops — data sources bound to `etch/loop` blocks

**Buffered** — persist with `await etch.saveAsync()`.

```ts
getAll(): Record<string, EtchLoop>;
add(loop: EtchLoop): string;              // returns generated id
update(loopId: string, loop: EtchLoop): void; // replaces the definition
delete(loopId: string): void;
findLoop(query: string): (EtchLoop & { id: string })[]; // fuzzy search by name/key
setForBlock(blockId: string, loop: BlockLoopBinding): void;
```

```ts
interface EtchLoop {
  key: string;      // stable, human-authored key
  name: string;     // display name in the loop manager
  global: boolean;  // reusable vs local to this document
  config: EtchLoopConfig;
  [key: string]: unknown;
}
type EtchLoopConfig =
  | { type: "wp-query"; args: WpQueryArgs }
  | { type: "wp-terms"; args: WpTermsArgs }
  | { type: "wp-users"; args: WpUsersArgs }
  | { type: "main-query"; args: WpQueryArgs }
  | { type: "json"; data: unknown[] };
```

```js
const loopId = etch.loops.add({
  key: "recent-posts", name: "Recent posts", global: true,
  config: { type: "wp-query", args: { post_type: "post", posts_per_page: 6, orderby: "date", order: "DESC" } },
});
etch.loops.add({ key: "tabs", name: "Tabs", global: false, config: { type: "json", data: [{ label: "One" }, { label: "Two" }] } });
await etch.saveAsync();
```

### Binding a loop to a block

```ts
interface BlockLoopBinding {
  loopId?: string;
  target?: string;               // dynamic-path form of "what to iterate"
  itemId?: string;                // variable name for current item
  indexId?: string;               // variable name for current index
  loopParams?: Record<string, unknown>;
}
```

```js
const [blockId] = etch.blocks.find({ type: "etch/loop" });
etch.loops.setForBlock(blockId, { loopId, itemId: "post", indexId: "i", loopParams: { count: 3 } });
```

### WP query args (`WpQueryArgs`) — mirrors `WP_Query`, open-ended (`[key: string]: unknown`)

Common typed fields: `post_type`, `posts_per_page` (`-1` = all), `offset`, `paged`,
`page` (alias of paged), `orderby` (`date|title|menu_order|rand|ID|author|name|
modified|parent|comment_count|...`), `order` (`ASC|DESC`), `post_status`,
`ignore_sticky_posts`, `author`, `author_name`, `category`, `category_name`, `tag`,
`tax_query`, `meta_query`, `s` (search keyword).

```ts
interface TaxQueryItem {
  taxonomy: string; field: "term_id"|"slug"|"name";
  terms: string | number | Array<string|number>;
  operator?: "IN"|"NOT IN"|"AND"; include_children?: boolean;
}
interface MetaQueryItem {
  key: string; value: string|number|Array<string|number>;
  compare?: "="|"!="|">"|">="|"<"|"<="|"LIKE"|"NOT LIKE"|"IN"|"NOT IN"|"BETWEEN"|"NOT BETWEEN"|"EXISTS"|"NOT EXISTS";
  type?: "NUMERIC"|"BINARY"|"CHAR"|"DATE"|"DATETIME"|"DECIMAL"|"SIGNED"|"TIME"|"UNSIGNED";
}
```

```js
etch.loops.add({
  key: "featured-news", name: "Featured news", global: false,
  config: { type: "wp-query", args: {
    post_type: "post",
    tax_query: [{ taxonomy: "category", field: "slug", terms: ["news"] }],
    meta_query: [{ key: "featured", value: "1", compare: "=" }],
  }},
});
```

`WpTermsArgs`: `taxonomy`, `orderby` (`name|slug|term_group|term_id|description|
count|...`), `order`.
`WpUsersArgs`: `role`, `include`, `exclude`, `search`, `search_columns`, `orderby`
(`ID|display_name|name|user_login|user_email|user_registered|post_count|
meta_value|meta_value_num|...`), `order`, `number`, `offset`, `paged`.

### Parameterized args

Numeric/boolean args accept loop-parameter expressions so one definition can be
reused with different `loopParams`:

```
type LoopParamRef = `$${string}`;                                  // e.g. "$count"
type NumericParam = number | LoopParamRef | `$${string} ?? ${number}`;
type BooleanParam = boolean | 0 | 1 | LoopParamRef | `$${string} ?? ${boolean}`;
```

```js
// posts_per_page falls back to 10 when `count` isn't supplied
{ type: "wp-query", args: { post_type: "post", posts_per_page: "$count ?? 10" } }
```

## etch.fields — custom field groups + per-post values

**Persists immediately** — every method is `*Async`; no `saveAsync()`.

### Group management

```ts
listGroupsAsync(): Promise<Record<string, CustomFieldGroup>>;
getGroupAsync(groupId: string): Promise<CustomFieldGroup>;
createGroupAsync(definition: CustomFieldGroup): Promise<string>; // returns id
updateGroupAsync(groupId: string, definition: CustomFieldGroup): Promise<void>;
deleteGroupAsync(groupId: string): Promise<void>;
```

```ts
interface CustomFieldGroup {
  label: string; description?: string;
  fields: CustomField[];
  assigned_to: CustomFieldAssignment;
  [key: string]: unknown;
}
type CustomFieldAssignment =
  | { post_types: string[]; op: "isIn"|"isNotIn" }
  | { post_ids: number[]; op: "isIn"|"isNotIn" }
  | { taxonomies: string[]; op: "isIn"|"isNotIn" };
```

```js
const groupId = await etch.fields.createGroupAsync({
  label: "Event details",
  assigned_to: { post_types: ["event"], op: "isIn" },
  fields: [
    { label: "Location", key: "location", type: "text" },
    { label: "Capacity", key: "capacity", type: "number", required: true },
  ],
});
```

### Field management

```ts
addFieldAsync(groupId: string, field: CustomField): Promise<void>;
updateFieldAsync(groupId: string, fieldKey: string, field: CustomField): Promise<void>;
removeFieldAsync(groupId: string, fieldKey: string): Promise<void>;
```

```ts
interface CustomField {
  label: string; key: string; type: CustomFieldType;
  description?: string; required?: boolean; [key: string]: unknown;
}
type CustomFieldType = "text" | "textarea" | "number" | "boolean" | (string & {});
```

### Field values (per post)

```ts
getValuesAsync(postId: number): Promise<PostCustomFieldValuesResponse>;
getValueAsync(postId: number, fieldKey: string): Promise<PostCustomFieldValueResponse>;
setValueAsync(postId: number, fieldKey: string, value: unknown): Promise<PostCustomFieldValueResponse>;
setValuesAsync(postId: number, values: Record<string, unknown>): Promise<void>;
deleteValueAsync(postId: number, fieldKey: string): Promise<void>;
```

```js
const { field } = await etch.fields.getValueAsync(42, "location");
await etch.fields.setValueAsync(42, "location", "Berlin");
await etch.fields.setValuesAsync(42, { capacity: 500, sold_out: false });
const all = await etch.fields.getValuesAsync(42);
for (const [groupId, group] of Object.entries(all.groups)) {
  console.log(groupId, group.label, group.fields);
}
await etch.fields.deleteValueAsync(42, "location");
```

## etch.navigation — move around the builder, switch post/template

```ts
goTo(place: NavigationPlace): void;
getCurrentPlace(): NavigationPlace;
getPlaces(): NavigationPlace[];
openPostAsync(postId: number): Promise<void>;
openTemplateAsync(templateId: number): Promise<void>;
getActivePostId(): number | null;
isEditingTemplate(): boolean;
listPostsAsync(postType?: string): Promise<PostSummary[]>;
listTemplatesAsync(): Promise<TemplateSummary[]>;
```

```
type NavigationPlace = "builder"|"templates"|"content-hub"|"style-manager"|"loop-manager"|"asset-manager";
```

```js
etch.navigation.goTo("style-manager");
etch.navigation.getCurrentPlace(); // "style-manager"
etch.navigation.goTo("builder");

const pages = await etch.navigation.listPostsAsync("page");
await etch.navigation.openPostAsync(pages[0].id);
console.log(etch.navigation.getActivePostId(), etch.navigation.isEditingTemplate());
```

## etch.ui / etch.history

```ts
// etch.ui
getColorScheme(): "light"|"dark";
setColorScheme(scheme: "light"|"dark"): void; // persisted locally
toggleColorScheme(): void;
isInterfaceHidden(): boolean;
setInterfaceHidden(hidden: boolean): void;
toggleInterface(): void;
exitToWordPress(): void; // leave the builder, return to wp-admin

// etch.history — same undo/redo stack as the keyboard shortcuts;
// scripted mutations participate automatically
undo(): void;
redo(): void;
canUndo(): boolean;
canRedo(): boolean;
```

```js
etch.blocks.setText(id, "Draft");
if (etch.history.canUndo()) etch.history.undo(); // reverts the setText above
```
