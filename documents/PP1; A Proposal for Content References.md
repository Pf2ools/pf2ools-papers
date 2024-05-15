|  **Title:** | A Proposal for Content References |
| ----------: | --------------------------------- |
| **Author:** | Spappz                            |
|   **Date:** | 2024-05-14                        |

---

# References and modifications

This proposal introduces `reference`, a new top-level content property. It is designed to describe provide a versatile and easily implementable structure to represent when content references (or is referenced by) other content. It can be considered analogous to Giddycode's `_copy` with improved syntax, nuance, and extensibility. Methods and mechanisms for handling references in the Pf2ools App are also described.

- [A reference to `reference`](#a-reference-to-reference)
  - [Putting `target` on target](#putting-target-on-target)
  - [How `modifications` modifies](#how-modifications-modifies)
    - [Primitive `Modification` `type`s](#primitive-modification-types)
  - [The types of `type`](#the-types-of-type)
    - [Reprint hierarchy](#reprint-hierarchy)
- [Programming considerations](#programming-considerations)
  - [Loading `reference`s](#loading-references)
  - [Unloading `reference`s](#unloading-references)
  - [Homebrew and related silliness](#homebrew-and-related-silliness)
    - [Problems with loading](#problems-with-loading)
      - [NewSource is loaded first](#newsource-is-loaded-first)
      - [OldSource is unloaded first](#oldsource-is-unloaded-first)
    - [Problems with listing](#problems-with-listing)
    - [Problems with rendering](#problems-with-rendering)
      - [`reprint` and `variant`](#reprint-and-variant)
      - [`replacement` and `extension`](#replacement-and-extension)
      - [The `data` doesn't exist but we still need rendering!](#the-data-doesnt-exist-but-we-still-need-rendering)
  - [What about `tags`?](#what-about-tags)
  - [Edge cases and other considerations](#edge-cases-and-other-considerations)
    - [Arbitrary `reference` chains](#arbitrary-reference-chains)
    - [Generalised reprint hierarchy](#generalised-reprint-hierarchy)
  - [Pseudocode summary](#pseudocode-summary)
    - [The source-loading process](#the-source-loading-process)
    - [The source-unloading process](#the-source-unloading-process)
    - [Generating the left-hand list](#generating-the-left-hand-list)
    - [Rendering statblocks](#rendering-statblocks)
- [Further work](#further-work)
  - [Multiple simultaneous `replacement`s](#multiple-simultaneous-replacements)
  - [Toggle overwritten data](#toggle-overwritten-data)
  - [Errata management](#errata-management)
    - [Remaster management](#remaster-management)
  - [Tangential systems](#tangential-systems)
    - [Source dependencies](#source-dependencies)
    - [View-changing sources](#view-changing-sources)

## A reference to `reference`

The property is called `reference` because, at the most abstract level, it describes a _reference_ from one statblock to another. The general structure is as follows:

```ts
type Reference = {
	type: "reprint" | "variant" | "replacement" | "extension";
	target: referenceTarget;
	modifications?: Modification[];
};
```

These three properties are sufficient in describing what the reference semantically represents: what _kind_ of reference is being made, what is actually _being_ referenced, and what _changes_ are being applied to that reference (if any).

### Putting `target` on target

Let's start simple.

`target` is just a unique determination for the original, referenced data. It uses the same format as all other `referenceTarget`s already used in the schema:

```ts
type referenceTarget = {
	name: string;
	sourceID: string;
	specifier?: string;
};
```

The datatype is always assumed to be the same as the content itself.

### How `modifications` modifies

The optional `modifications` property contains an array of `Modification`s (wow!). A `Modification` is essentially an 'atomic' instruction, for instance "add 2 to the HP" or "remove the 'Bite' action". Each `Modification` in the array is applied sequentially, in order, one-by-one.

For simplicity the schema won't ensure that the `Modifications` make sense (e.g. if you try to multiply an object by 2), but there will be a test script to do so. So at least the repo will be safe!

```ts
type Modification = {
	type: "types" | "to" | "be" | "determined" | "later";
	target: {
		property: string;
		name?: string; // used to target specific array elements
	};
	value?: NonNullable<any>; // this doesn't work actually in TS but you get the idea
	parameters?: Record<string, NonNullable<any>>; // form dependent on `type`
};
```

> [!NOTE]
> `data` may exist simultaneously as `reference`, but not at the same time as `reference.modifications`. This is for three reasons:
>
> 1. Keeps the schema clean (no conditional property-optionality!)
> 2. It reflects how statblocks are actually written: either it's "as _Statblock_ (SRC), but ..." (i.e. `reference.modifications`) or it simply presents a whole new statblock (i.e. `data`).
> 3. The assumption that `data`, where it exists, is always 'complete' (i.e. fully renderable) allows us to simplify quite a bit of programming, as we'll see [below](#programming-considerations).

#### Primitive `Modification` `type`s

There will likely be many `type`s of `Modification` added as required, but there are some obvious 'primitive' functions that are worth implementing from the outset.

- **Definition:** Define the target property, overwriting any data that already exists there.
- **Addition:** Add to the target number (also covers subtraction).
- **Multiplication:** Multiply the target number (also covers division).
- **Insertion:** Append the value onto the end of the target array.
- **Deletion:** Set target property as `undefined`, or delete the target element from an array.

Some more complex `type`s can be easily defined using composition. For instance, a 'replacement' can be made of two `Modification`s: first a `deletion`, and then an `insertion`. We can investigate the _complete_ list of `type`s as we work through the data.

Lastly, some `Modification` `types` are restricted based on the `reference` `type`. For instance, there is no reason an `extension` should _ever_ delete data.

### The types of `type`

Now we tackle the actually complicated part. There are four `type`s of `reference`, each reflecting how the reference functions.

1. **`reprint`:** The statblock is (nearly) identical to another one, and it's _intended_ to be identical.
   - `modifications` are permitted for non-impactful wording differences, if they aren't errata-ed out.
   - We define one source as the 'most significant version' (see [Reprint Hierarchy](#reprint-hierarchy) below), and display only that one in the left-hand list. If that source is filtered out, but another source with a reprint is allowed, the statblock may be shown under the latter source (Giddycode's "Include References" method may be used).
   - If `modifications` exist, it should be possible to see each version, preferably hidden behind a button.
   - A low-priority footer or button can indicate other sources the content is found in (e.g. "reprinted in SoG2 p51, LOME p192").
2. **`variant`:** The statblock is dependent on, but distinct to, the target (e.g. unique NPCs based on generic monsters).
   - These are displayed in the left-hand list as entirely separate content.
   - A low-priority footer or button can indicate the original source material (e.g. "based on {@creature air mephit}").
3. **`replacement`:** The statblock replaces the target entirely everywhere. This should be used primarily for homebrew revisions.
   - A button should reveal the original, either in-line or as a pop-up.
   - A setting could be introduced to allow selective control of which `replacement`s are applied. This is certainly possible using the set-up described [below](#toggle-overwritten-data), but it's out-of-scope for now.
4. **`extension`:** The statblock adds additional content to the target but doesn't supersede or replace it (e.g. a new option for the _animal form_ spell).
   - Only the original item should be displayed in the left-hand list.
   - When a target statblock is viewed, the extended elements should be merged in, ideally with some UI to indicate it as being 'foreign' content ([example](../assets/PP1/extension-replacement-formatting-example.png) piggy-backing off `@homebrew` formatting).

For all types except `reprint`, the reference should be _explicit_ in the text; this is not intended as a conversion shorthand (you can always copy-paste the file). A `reprint` can be inferred if it's reasonably obvious to a reasonable reader; otherwise, use `replacement` or `variant` (maintainers' discretion).

> [!NOTE]
> There are some restrictions placed depending on `type`:
>
> - No `reprint`s, `replacement`s, or `extension`s targetting content in the same source.
> - `replacement`s must either define `data` or use `modifications`.
> - `variant`s and `extensions`s must use `modifications` (i.e. they may not have `data` defined).
> - No cyclical `reference`s of any kind (not really related to `type` but worth stating).
>
> I can enforce all of these within the repo. If any occur in private brew, I'm satisfied in leaving the behaviour undefined; it's their problem for not using the test scripts!

#### Reprint hierarchy

Amongst `reprint`s, the 'most significant version' is determined with the following priority:

1. If one version is remaster content and the other is legacy, favour remaster unless the legacy setting is toggled.
2. If the 'source category' differs, favour sources in the following order: sourcebooks (CRB, APG, SoM, etc.), setting books (Lost Omens), adventures, miscellaneous Paizo stuff (comics, blog posts, etc.), then finally homebrew.
3. If both sources are in the same category, favour the earliest release date.

> [!NOTE]
> I'm doubling down on point 3 favouring the _earliest_ release date because that just makes the most sense. If something is printed in the Core Rulebook and also Book of the Dead, surely you'd want to see it predominately associated with the CRB?
>
> Yes, there was a [weird case with the Tattoo Artist feat](https://discord.com/channels/776300932928897024/949797222101438545/1188316007278837812). I'd chalk that up to Paizo being disorganised as always, but I am happy to nag them personally if they forget to errata things. We're not hiding from them like mothersite, so it'll be mutually beneficial if we act as an 'accountability' measure (i.e. unpaid QA editors)—presumably Paizo want to be consistent too!

## Programming considerations

This section is going to be quite long, but it's where I lay out the exact way I'm envisaging this data being applied in the app. There might be other viable methods, but, rather than ponder the Platonic form of PF2 statblocks in a vacuum, I devised these systems so they should work _immediately_, no matter what sort of crazy `reference` trees are going into the data. So, please bear with me—I'm just making sure you're not being dropped in the deep end head-first! 😊

Throughout this section, I'm going to refer to some dummy data: NewContent in NewSource, and OldContent in OldSource. NewContent, in particular, is a `reference` to OldContent.

### Loading `reference`s

Let's imagine we have OldSource loaded, and we want to load NewSource. As we're iterating through NewSource, adding its content into memory, we get to NewContent, notice its `reference`, and pause. We now perform two steps: inflate and point.

The information to render a statblock is stored in `data`. If NewContent already has `data` defined, then we don't need to inflate anything—NewContent can already be rendered. However, if it doesn't, we need to _inflate_ NewContent's `data` property by looking at OldContent and merging in NewContent's `modifications` (if any). Once this is done, we simply attach it onto NewContent by defining its `data` property in memory. (Note: this is guaranteed to be non-destructive due to the exclusivity of `data` and `modifications` in the repo.) With `data` now ensured to exist on NewContent, we don't need to fiddle with `data` again; it can be rendered no matter what happens to OldContent.

But what do we do with OldContent? After all, the app might need to change how it displays OldContent—at the very least, it probably needs a footer/button exposing its connection to NewContent. This is where `pointers` comes into play. `pointers` is an app-internal property attached onto data, which represents a _practical_ link between content, contrasting to `reference` which only defines how data is _defined_.

`pointers` is itself an array of `Pointer` objects.

```ts
type Pointer: {
	type: "reprint" | "variant" | "replacement" | "extension", // identical to `reference.type`
	target: referenceTarget,
	incoming?: boolean, // The direction of the reference
	hidden?: boolean, // Whether this content containing this Pointer should be hidden in the left-hand list
}
```

`Pointer`s should be _reciprocal_, meaning that both NewContent _and_ OldContent each get a `Pointer` representing their relationship. By establishing and obeying this reciprocity, we can skip a lot of error-checking and slow look-ups. We can just distill all the important information once, on load, when speed is unimportant, and rely on these trusted structures henceforth.

> [!NOTE]
> The `hidden` property's value can, in theory, be calculated from `type`, `incoming`, and information about the sources, but it's very useful to just have it defined from the outset. It saves us extra source look-ups on list-rendering and also the limits code complexity to this one-time process.

For illustration's purpose, let's run through each possible case. I encourage you to satisfy yourself that the `Pointer`s below correspond to how we want the data represented!

- If NewContent is a `reprint`, we need to calculate whether NewSource or OldSource is the 'most significant version'.

  - If NewContent is most significant, then it receives `{type: "reprint", target: OldContent}` and OldContent receives `{type: "reprint", target: NewContent, incoming: true, hidden: true}`.
  - If OldContent is most significant, then NewContent receives `{type: "reprint", target: OldContent, hidden: true}` and OldContent receives `{type: "reprint", target: NewContent, incoming: true}`.

  We also add a `Pointer`-pair for every _other_ `reprint`. This allows us to intelligently filter them (see [below](#generating-the-left-hand-list)). You can imagine this abstractly as all `reprint`s fundamentally _being_ the 'same entity', so they naturally all point to each other.
  The pointer tells the renderer to insert the 'reprint text' into the statblock; the value of `incoming` indicates whether the text should be "Reprint of ..." or "Reprinted in ...". If `modifications` exists, show a button to view the original. Also, the `hidden` property is overridden if `target` is filtered out.

- If NewContent is a `variant`, then it receives `{type: "variant", target: OldContent}`, and OldContent receives `{type: "variant", target: NewContent, incoming: true}`. The pointer tells the renderer to insert the 'variant text' into the statblock: "Variant of {@dataRef OldContent}" for `incoming: false`; nothing otherwise. (Note: the reciprocal pointer is always present, even if it doesn't affect display!)

- If NewContent is a `replacement`, then it receives `{type: "replacement", target: OldContent}`, and OldContent receives `{type: "replacement", target: NewContent, incoming: true, hidden: true}`. Whenever OldContent would be rendered—whether in a list page or by an `@tag`—its pointer tells the renderer to instead look up and render NewContent. The 'view original' button obviously overrides the pointer and correctly shows OldContent (this can be achieved using a flag on the button inaccessible via normal `@tag`s).

- If NewContent is an `extension`, then it receives `{type: "extension", target: OldContent, hidden: true}` and OldContent receives `{type: "extension", target: NewContent, incoming: true}`. This is the special case: whenever OldContent would be rendered, its pointer tells the renderer to replace it with NewContent's `data`, but _preserve_ OldContent's `name` and `source` for display; 'extension text' naturally be added based on the `pointer` too.
  - For NewContent's `modifications` to be indicated in the UI, you will also need to 'tag' the changes with extra properties so the renderer knows how to highlight them. This will probably be tricky; I have some half-baked ideas but none of them are particularly nice. If it's not feasible at this stage, so be it 🥴

The general idea throughout is that, by adding one 'check pointers' step at the start of the render process, and one special, always-on filter in list pages, you can hide and redirect statblocks as appropriate, _without_ needing a separate state-tracking store.

### Unloading `reference`s

When NewSource is unloaded, we once again iterate through its content. Note: _only_ its content, since we've ensured that all pointers are reciprocal! Once we find NewContent with its `pointers`, we iterate through that, and delete each corresponding pointer on OldContent. We then delete NewContent from the main content store as normal.

Simple!

### Homebrew and related silliness

The existence of homebrew requires some extra considerations, since we can no longer guarantee that either NewContent or OldContent are valid or even exist.

#### Problems with loading

If both OldSource is homebrew, there is a possibility that it isn't loaded. When this occurs, I'll term NewContent an 'orphan', in that its 'parent' (OldContent) doesn't currently exist in the app. Orphans can occur in two ways.

##### NewSource is loaded first

If the OldSource is in the repo, it can simply be (temporarily) fetched and loaded so that NewContent's `data` can be built. Once this is done, NewContent can be rendered even if it is once again orphaned by the (automatic) unload of OldSource.

If the source isn't in the repo, however, we have a bit of a problem. There are two easy options, each unideal for different reasons. Most easily, we could just reject the entire load, but this will probably be bad UX and would dissatisfy people who prefer something rather than nothing. We could alternatively _only_ reject the orphans, and load the remaining content, but this would create another poor UX situation: loading OldSource followed by NewSource produces a different app-state than loading NewSource followed by OldSource! I'm not sure we can trust the average user to recognise (or accommodate for) this significance.

What we need is some way to 'stash' orphans until they're 'adopted'. That is, when OldSource is loaded, it can _automatically_ match up to the orphaned NewContent, inflate its `data`, and build reciprocal `Pointer`s automatically. To achieve this, I introduce a new store, which I'll call `UnloadedContent`. It contains an array of `UnloadedContentReference`s.

```ts
type UnloadedContentReference = {
	type: string; // "feat", "spell", etc.
	sourceID: string;
	name: string;
	specifier?: string;
	orphans: referenceTarget[]; // The already loaded content that wants to reference this unloaded content
};
```

When we load NewContent and find that its `reference` targets something that doesn't (yet) exist, we add that _target_ to `UnloadedContent`. So, in the above case, we add an `UnloadedContentReference` representing OldContent into `UnloadedContent`, and put NewContent in its `orphans`.

Next, on any _subsequent_ source-load, we iterate through `UnloadedContent` and check whether any content there matches. If it turns out we're loading OldContent, we can build the `Pointer`s between NewContent and OldContent, inflate NewContent's `data`, and then delete it from `UnloadedContent`. Since `reference`s are already rare, and `reference`s between private homebrew are even rarer, this won't represent a substantial computational load even in the worst case (plus, it's _only_ on source-load).

> [!TIP]
> Remember: `UnloadedContent` isn't a list of orphans, it's a list of content we're waiting for—the 'parents' or the orphans. Even if we have 1000 orphaned statblocks dependent on OldContent, `UnloadedContent` will only contain one element.

##### OldSource is unloaded first

Another way to create an orphan (in theory) is to _remove_ its parent. However, since we've already attached the 'inflated' `data` to NewContent, this doesn't actually matter for most cases!

So, we start doing everything as normal. On unload of OldSource, we iterate through and delete its content as normal, until we hit OldContent. OldContent has a `Pointer`, so we also remove the corresponding `Pointer` on NewContent. We _also_ now check whether OldContent's `Pointer`'s `incoming` property is truthy (that is, that NewContent `references` _it_, not the other way around). If so, we do one extra thing: add an `UnloadedContentReference` for OldContent to `UnloadedContent`, and populate its `orphans` with NewContent.

Now, if OldSource is reloaded, we can automatically rebuild the `Pointer`s between OldContent and NewContent. Not too important for `variant`s, but very important for `replacement`s!

#### Problems with listing

Next, we have to consider how the left-hand list interacts with potentially orphaned content. Previously, we relied on the `hidden` property in the `Pointer`s, but these are now absent, so we need a new heuristic. Luckily, this is pretty simple:

- Show `reprint`s and `variant`s
- Hide `replacement`s and `extension`s

This follows from just the _meaning_ of the `reference`. A `reprint` or `variant` indicates a 'genealogical' link; NewContent derives from OldContent, but is a wholly new thing. We might not know where this link goes (because OldSource is unloaded), but we don't _need_ to know to see the new thing!

On the other hand, a `replacement` or `extension`'s link is not merely 'genealogical', it's one of _identity_: it's stating that NewContent _is_ OldContent, and OldContent _is_ NewContent, so, if OldContent doesn't exist, then neither does NewContent. (Yes, even if `data` is inflated so we could plausibly display it; a user doesn't typically want to see NewSource's version of OldContent if they don't use OldSource in the first place.)

> [!NOTE]
> Why not just hide all orphans? The reason is that the left-hand list should, ideally, show _all_ applicable content given a set of filters, not just content you happen to have readily accessible. If your filters are "anything in X source", you should see what's in X source—even if you might not be able to render it right now! I'd personally view any alternative as a ~~lie~~ misrepresentation to the user.

#### Problems with rendering

Lastly, let's look at rendering orphaned statblocks. Our method depends on the `reference` `type`.

##### `reprint` and `variant`

`reprint`s and `variant`s, once inflated, can stand independently. An orphaned `reprint` is just the only print (like 99.9% of data); an orphaned `variant` is just a statblock without alternatives/originals. For in-repo brew, we're also assured that `data` is always inflated, even if OldSource isn't loaded.

The only issue lies with the "Reprint of {@tag ...}" (and equivalent) text, and the "View Original" (and equivalent) button, since they're direct references to unloaded data. This is handled [below](#the-data-doesnt-exist-but-we-still-need-rendering), but do note that the _raw rendering_ of NewContent in this case doesn't require OldSource to exist.

##### `replacement` and `extension`

As above, an orphaned `replacement` or `extension` should be excluded from the left-hand list. This means it can only ever be rendered due to an `@tag`.

The solution here is simple: ban it. Seriously—I can only come up with _one_ legitimate case where NewSource might want to include an `@tag` to the original, unadulterated statblock, and that's when you've got book text like:

> Although the {@spell fireball|CRB||showOriginal=true} spell deals fire damage in the _Core Rulebook_, in Settinglandia {@spell fireball|CRB} deals positive damage.

And I'm happy for that hyper-specific case to simply lose the first tag. In any other case, you would always simply tag the referenced statblock and let the app apply the redirection. If the `replacement` or `extension` exists, you should want to use it!

I've brainstormed a way to validate literally every tag in every file in the repo, so with this script added to `npm run test` the problem should, again, only exist in malformed private brew.

> [!NOTE]
> If you _want_ to support `showOriginal=true` or whatever other silliness can be conceived via `@tag`s, be my guest. There is the _very rare_ case where both OldSource and NewSource are homebrew, and OldContent is _already_ a replacement, so you might have:
>
> > Although in _Settinglandia: The First Place_ the {@spell fireball|CRB||showReplacement=SL1} spell deals positive damage, in _Settinglandia: The Second Place_ {@spell fireball|CRB||showReplacement=SL2} deals holy damage.
>
> But, again, I'm happy to simply dismiss this as gratuitous.

##### The `data` doesn't exist but we still need rendering!

Let's enumerate all of our systems' holes to see when this occurs:

- NewContent references OldContent. NewSource is loaded, but OldSource has never been. The user clicks on an `@tag` somewhere in NewSource that leads to the OldContent.
- NewSource and OldSource are both private, and both loaded. The user unloads OldSource, and then clicks on an `@tag` somewhere in NewSource leading to the OldContent.
- NewSource and OldSource are both private, and NewContent is a (most significant) `reprint` or `variant` of OldContent. OldSource was never loaded, and the user ignored the warnings resulting from this. The user then tries to view NewContent.
- NewSource and OldSource are both private, and NewContent is a `replacement` or `extension` of OldContent. OldSource was never loaded and the user ignored the warnings resulting from this. The user then tries to click an `@tag` somewhere in NewSource leading to the NewContent. NewSource's converter has also never run the test scripts, or has chosen to ignore them.

In every case, we just render a placeholder statblock: "This content depends on data that hasn't been loaded." If the source is in the repo, you can provide an option to load it, but nothing more needs to be done. This is now an excessive number of edge-cases deep, primarily compromised of malformed private brew—just call it a day!

### What about `tags`?

In general, the `tags` of a given piece of content won't change much when `reference`d. Therefore, `tags` can simply be accessed via `modifications` like any other property in `data`.

But wait—aren't `tags` and `data` both top-level properties? Not any more! 😏

> [!NOTE]
> I can justify this to myself by shifting my abstraction of `data` from 'the statblock' to 'the payload'. Now, you've got `type`, `name`, and `source`—all properties that define a UUID (and associated metadata)—plus `data`, which is just... all the data.
>
> This also makes the schema marginally simpler, so hooray for that?
>
> I might rename `tags` to `_tags` though, just to make it abundantly clear that it's not render-data.

### Edge cases and other considerations

This is the part where I call out some flaws.

#### Arbitrary `reference` chains

You may be wondering what happens when brewers (or Paizo) start replacing extensions of variants of variants of variants, in an arbitrary chain of `reference`s that can't be predicted ahead of time. I thought a lot about this too 😩

The `Pointer` system is 'decentralised', in that neither the app nor the repo require a single object to be the authoritative source on how content relates to each other. Data is simply handled as it appears, meaning that, _as far as I can tell_, all of the above systems can perfectly handle arbitrary `reference` chains—with two exceptions.

1. **Overwrite conflicts.** What happens if two sources both want to extend the same content? The present set-up doesn't allow the app to merge multiple `extension`s on the same data—_chains_ of `extension`s yes, but not branching trees. If two sources both want to replace the same content, there's also no easy way the app can know which one the user prefers. My solution? Display them both! I would naturally just render the statblocks side-by-side, but a button to 'switch versions' works too if you can be bothered with that. I use this approach below in the [pseudocode summary](#pseudocode-summary).
   Alternatively, just render an 'error-message' statblock, like "render conflict: source1 and source2 are both trying to overwrite the same content; please unload one of the sources". It's an exceptionally niche case, and one I'm happy waving away.
2. **`extension`s of demoted `reprint`s.** Let's imagine Paizo release a feat as part of an AP. Let's say someone makes an `extension` of this feat. Paizo later reprint this feat in a sourcebook. Since the sourcebook's `reprint` supersedes the AP's, the latter is normally hidden from the left-hand list. As a result, unless you access the AP's feat directly (e.g. omnisearch), the `extension` to the feat effectively disappears.
   Besides the natural 'solution' of "brew-maintainer problem; sort it out yourself", I could fiddle with a script for Github Actions so an alert is raised if a PR would 'invalidate' data like this. If there's a more elegant solution in the way `Pointer`s fundamentally work, I'm all ears.

> [!NOTE]
> `replacement`s and `variant`s of `reprint`s don't have the same problem as with `extensions`, since they 'fork' the content into a new element in the left-hand list. Also note that, with the reprint hierarchy favouring _earlier_ releases, this isn't a problem if Paizo reprint content within the same 'source category'.

#### Generalised reprint hierarchy

The 'source category', as mentioned in the [reprint hierarchy](#reprint-hierarchy), isn't really something that can be well-applied outside Paizo's very particular publishing style, so I'm hesitant to just stick in a property for it. Is there _really_ a difference between a 'sourcebook' and a 'setting book' if the former almost always refers to Golarion anyway? What if Paizo stop the Lost Omens line but still publish about Golarion? What if Paizo release a 10-page, setting-neutral softcover—is that a 'sourcebook' on par with the CRB or just 'miscellaneous' like a blog post? We should probably learn from mothersite's source-categorising hell.

Ideally, the schema should be universal, and it'd be nice if homebrew sources were tagged in a way consistent with Paizo. This suggests there should be some 'objective' categories about the actual product, and the reprint hierarchy is read _through_ these properties (e.g. 'sourcebooks' are always 'big books'). The schema already has some ('comic' and 'blog'), but I'm not sure 'big book' is a good descriptor. So we'll have to think hard on how to actually implement this.

Worst come to worst, I could simply maintain a manually ordered list of sources in the app itself, and simply fall back to 'maintainer discretion' as an excuse when there's ambiguity.

### Pseudocode summary

This _very nearly last_ section serves two purposes:

- Summarise everything above
- Restate the important bits in another language, in case I didn't make sense the first time

I make some black-box assumptions about how the app works, but it should be sufficient for you to get the gist of the _logic_. The algorithms can probably be optimised to shave off a few hundred nanoseconds, but I'm favouring clarity over efficiency for illustration. I'm also using basic objects and arrays over maps and sets for the same reason (even if a few things _really should_ be the latter).

#### The source-loading process

This is the workflow run for each new bit of data loaded during a broader source-/bundle-loading workflow. Functions are split out for readability (not including error-handling). This is the most complex stage though!

```js
ContentManager.loadData = function (datatype, newData) {
	// If there's no `reference` silliness, just load as normal
	if (!newData.reference) return ContentManager.addData(datatype, newData);

	// Is the referenced data loaded?
	let oldData = ContentManager.getContent(datatype, newData.reference.target);
	// If not, try to fetch it from the repo
	if (!oldData) oldData = DataRepo.getContent(datatype, data.reference.target);
	// Let's start by assuming we've found oldData
	if (oldData) {
		// First, inflate newData's `data` if it doesn't already exist
		if (!newData.data) {
			newData.data = StatblockManipulator.applyModifications(oldData, newData.reference.modifications);
		}
		// Do the load!
		ContentManager.addData(datatype, newData);
		// Next, set up the pointers (function described below)
		ContentManager.applyPointers(newData, oldData);
	} else {
		// But wait, what if we can't find oldData? Time to make do... (see below)
		ContentManager.addData(datatype, newData);
		UnloadedContent.addUnloadedDataFromOrphan(newData);
	}

	// One last thing: were we already waiting on newData to adopt some orphans?
	for (i = 0; i < UnloadedContent[datatype].length; i++) {
		if (
			UnloadedContent[datatype][i].sourceID === newData.source.ID &&
			UnloadedContent[datatype][i].name === newData.name.primary &&
			UnloadedContent[datatype][i].specifier == newData.name.specifier
		) {
			// We've found that newData can adopt some orphans!
			UnloadedContent[datatype][i].orphans.forEach((orphan) => {
				const adoptedOrphan = ContentManager.getContent(datatype, orphan);
				if (!adoptedOrphan.data) {
					adoptedOrphan.data = StatblockManipulator.applyModifications(
						newData,
						adoptedOrphan.reference.modifications,
					);
				}
				ContentManager.applyPointers(adoptedOrphan, newData);
			});

			// Finally, remove that UnloadedContentReference
			UnloadedContent[datatype].splice(i, 1);
			return;
		}
	}
};
```

```js
ContentManager.applyPointers = function (newData, oldData) {
	// Initialise
	if (!newData.pointers) newData.pointers = [];
	if (!oldData.pointers) oldData.pointers = [];

	// If newData is a reprint, we need to add a pointer to it to all other reprints.
	if (newData.reference.type === "reprint") {
		// We do this by traversing oldData's `reprint` pointer-tree, building a list, and then iterating through it.
		// Building the list first rather than adding the pointers as you walk prevents you getting stuck in a loop or adding duplicates.
		const findAllReprints = (data, reprints = []) => {
			if (data.reference.type === "reprint") reprints.push(ContentManager.getUUID(data));

			if (data.pointers) {
				data.pointers
					.filter((pointer) => pointer.type === "reprint" && !reprints.includes(ContentManager.getUUID(pointer)))
					.forEach((pointer) => traverseReprintTree(ContentManager.getContent(pointer)));
			}

			return reprints;
		};
		const allReprints = findAllReprints(oldData);

		allReprints.forEach((cousinUUID) => {
			const cousin = ContentManager.getByUUID(cousinUUID);
			const [newDataPointer, cousinPointer] = ContentManager.getPointers(newData, cousin);
			newData.pointers.push(newDataPointer);
			cousin.pointers.push(cousinPointer);
			ContentManager.updateData(newData.type, ContentManager.UUIDToReference(cousinUUID), cousin);
		});
	}

	// Now we build the actual `Pointer`s between NewData and OldData
	const [newDataPointer, oldDataPointer] = ContentManager.getPointers(newData, oldData);
	newData.pointers.push(newDataPointer);
	oldData.pointers.push(oldDataPointer);

	// Apply the changes to the referenced pair
	ContentManager.updateData(oldData.type, ContentManager.getReference(oldData), oldData);
	ContentManager.updateData(newData.type, ContentManager.getReference(newData), newData);
};
```

```js
UnloadedContent.addUnloadedDataFromOrphan = function (orphan) {
	const datatype = orphan.type;
	const unloadedTarget = orphan.reference.target;
	const orphanReference = ContentManager.getReference(orphan);

	// Check if the unloaded content is already registered and, if so, simply add the orphan to its list
	for (unloadedContent of UnloadedContent[datatype]) {
		if (
			unloadedContent.sourceID === unloadedTarget.sourceID &&
			unloadedContent.name === unloadedTarget.name &&
			unloadedContent.specifier == unloadedTarget.specifier
		) {
			unloadedContent.orphans.push(orphanReference);
			return;
		}
	}

	// If the unloaded content is new, create a new entry
	UnloadedContent[datatype].push(new UnloadedContentReference(unloadedTarget, [orphanReference]));
};
```

#### The source-unloading process

This is the workflow that runs whenever a piece of data is *un*loaded.

```js
ContentManager.unloadData = function (data) {
	// If the data has no `reference` or `pointers`, then the data is independent to everything else and we can freely delete it (this applies for most stuff)
	if (!data.reference && !data.pointers) return ContentManager.deleteData(data);

	// Next we need to check whether the data was waiting on some UnloadedContent.
	// This only happens if the data has a `reference` and no `pointers` (function described below).
	if (data.reference && !data.pointers) return UnloadedContent.removeByOrphan(data);

	// If we're here, then we're certain that the data has `pointers`. We now need to remove them.
	ContentManager.removeReciprocalPointers(data);
	// And with that done, we can now safely delete the data itself.
	return ContentManager.deleteData(data);
};
```

```js
ContentManager.removeReciprocalPointers = function (data) {
	// For each `pointer` on the data...
	data.pointers.forEach((pointer) => {
		// Look up the content to which it's pointing
		const pointedContent = ContentManager.getContent(data.type, pointer.target);
		// Then find that content's `pointer` which is reciprocal to `data`'s one (i.e. the `pointer` on `pointedContent` that points back to `data`)
		for (i = 0; i < pointedContent.pointers.length; i++) {
			const reciprocalPointer = pointedContent.pointers[i];
			if (
				reciprocalPointer.sourceID === data.source.ID &&
				reciprocalPointer.name === data.name.primary &&
				reciprocalPointer.specifier === data.name.specifier
			)
				// And delete it
				return pointedContent.pointers.splice(i, 1);
		}
	});
};
```

```js
UnloadedContent.removeByOrphan = function (data) {
	for (i = 0; i < UnloadedContent[data.type].length; i++) {
		const unloadedContent = UnloadedContent[data.type][i];
		for (j = 0; j < unloadedContent.orphans.length; j++) {
			const orphan = unloadedContent.orphans[j];
			if (
				orphan.sourceID === data.source.ID &&
				orphan.name === data.name.primary &&
				orphan.specifier === data.name.specifier
			) {
				// We've found the data in `unloadedContent`'s `orphans`! Now there's two things that can occur:
				// 1) It was the only orphan of that data, in which case we should delete the entire `unloadedContent`.
				if (unloadedContent.orphans.length === 1) return UnloadedContent[data.type].splice(i, 1);
				// 2) There are multiple orphans waiting on that data, in which case we should only remove that one orphan
				return unloadedContent.orphans.splice(j, 1);
			}
		}
	}
};
```

#### Generating the left-hand list

This is a big `.filter()` function to filter out content which shouldn't be displayed in the left-hand list. It's not actually quite so big once you remove the comments, but since it's a little counter-intuitive I want to be explicit about the logic going on.

```js
const displayedContent = ContentManager.getByDatatype(datatype)
	.filter((content) => {
		// Always show normal, stand-alone content with no pointers attached (this covers most stuff!)
		if (!content.pointers && !content.reference) return true;

		// Check for orphans
		if (!content.pointers && content.reference) {
			// Hide orphaned `replacement`s and `extension`s
			if (content.reference.type === "replacement" || content.replacement.type === "extension") return false;
			// Everything else is a `reprint` or `variant`, which we want to show (even if they can't currently be rendered)
			return true;
		}

		// If we've arrived here, then `content.pointers` is guaranteed to exist

		// Show anything with zero `hidden` `Pointer`s
		if (content.pointers.every((pointer) => !pointer.hidden)) return true;

		// If something has `hidden` `Pointer`s, we only want to *consider* displaying it if all of those `hidden` `Pointer`s are `reprint`s (since they have the edge-case where they might be displayed due to filtered sources)
		const hiddenPointers = content.pointers.filter((pointer) => pointer.hidden);
		if (hiddenPointers.some((pointer) => pointer.type !== "reprint")) return false;

		// If we've arrived here, then we're sure that the content is a `reprint` that *isn't* in the most-significant source
		// We also know that it won't be hidden for any reason other than its source significance
		// Therefore, we can safely *hide* it if either
		// 1) the 'show reprints' button isn't activated
		if (!UserFilters.showReprints) return false;
		// or 2) the 'show reprints' button is activated, but a more-significant source is being displayed
		if (hiddenPointers.some((pointer) => UserFilters.getActiveSources().includes(pointer.target.sourceID)))
			return false;
		// Note that the most-significant `reprint` is measurable by the fact that none of its `reprint` `Pointer`s make it `hidden`, which we've already dealt with above

		// Anything remaining at this point is a `reprint` as described, except the 'show reprints' button is activated and it's the most-significant source *among the active source-filters*! So, we now want to display it
		return true;
	})
	.filter(UserFilters.getFilterFunction());
```

#### Rendering statblocks

This function outputs an _array_ of statblocks because, presumably, if there is an arbitrary choice in how to resolve a statblock, you want to see each one (see [above](#arbitrary-reference-chains)). This should _only_ occur when multiple `replacement`s or `extension`s are applied to the same content; a rare case in a rare case, so if the floating embed looks ugly you can offload this as "stop loading stupid homebrew smh".

Some other function would wrap the output of `renderContent()` in the appropriate 'frame' (right-hand statblock on list pages, floating embed on mouse-hover, etc.).

```js
function renderContent(content) {
	// If there's no pointers, just act as normal.
	if (!content.pointers) return [doRenderForRealNow(content)];

	// However, if there are, we need to render all applicable ones.
	const statblockStack = [];

	content.pointers.forEach((pointer) => {
		// If the pointer doesn't require anything to happen, do nothing (wow).
		if (pointer.type === "reprint" || pointer.type === "variant") return;

		// Since it's illegal to directly tag `replacement`s and `extension`s, we can assume a redirection is required now.
		statblockStack.push(doRenderForRealNow(ContentManager.getContent(content.type, pointer.target)));
	});

	// If all the `Pointer`s were `reprint`s or `variant`s, then `statblockStack` is empty; we just render the statblock.
	if (!statblockStack.length) return [doRenderForRealNow(content)];

	// If not, then we have some number of statblocks already in `statblockStack`!
	return statblockStack;
}
```

## Further work

While brainstorming and working all this out, I came up with some things I had to set aside, lest I _literally never finish planning_ 😩

Here's some of those.

### Multiple simultaneous `replacement`s

A plausible situation may emerge where homebrew says "replace (weak) feats X and Y with one new, combined feat Z". The current system, however, only permits `reference`s to have one, single `target`. Although you can hack it in without changing the above with a replace-one-blacklist-others solution, it would unfortunately break `@tag`s.

One _could_ simply make `target` now `targets`, and contain an array of content to replace. However, this gets a bit messy since that obviously also affects the other `reference` `types`. I initially went down this route before deciding that there was probably a better way (or it could be patched in when required—the logic doesn't change much at all).

A way to recover consistency is to allow multiple `targets` in `variant`s and `extension`s. This would represent kinds of 'generating' statblocks. There are easy analogues:

- A multi-`target` `variant` is literally a (creature) template, generating one new statblock per `target` by applying the same `modifications` to multiple original statblocks.
- A multi-`target` `extension` is akin to a wide-ranging houserule. Imagine something like like "add the **blah** trait to X, Y, and Z"; this would be neatly represented by a single `extension` that `targets` X, Y, and Z simultaneously.

It would be very nice to have these! But there's a problem: they're completely incompatible with the current set-up whereby the renderer follows a `Pointer` chain to arrive at the final `data`. Presently, content can only _have_ one `data` representation; a multi-`target` `variant` would need one for _each_ of its `targets`. It would certainly be _possible_ to implement (e.g. the `variant` has an object mapping source data to an expanded `variant` representation), but that's some more complicated architecture stuff that needs to be decided early. It would be pretty bad to have this set up perfectly, only to later add logic to everything like "if multi-target, ignore everything below and instead use this bespoke method".

Moreover, a logical continuation from multi-`targets`, as well, is _descriptive_ targeting. Instead of choosing the `target` by name, you instead choose it via a description: "any creature with the dragon and unholy traits with either at least 60 HP or no fly speed". That sounds like a pretty cool feature, but it's suddenly... much more complicated to represent in data and handle in code. 😩

Something to think about.

### Toggle overwritten data

Some `references`, notably `extension`s and `replacement`s, 'overwrite' their source data insofar that the user sees it in the app. I can foresee that people will want to import homebrew with multiple `replacement`s/`extension`s but only actually _use_ a subset of them. Since a source is either loaded or not, they appear to have no recourse!

A UI solution would be to simply have a giant list of 'overwritten' data in the settings, each with checkboxes. A user can use this to toggle off (or back on) `replacement`s and `extension`s on a case-by-case basis. How to implement this in code?

I think we'll need a new store; let's call it `ViewModifications`, since it catalogues content which changes the user's view. `ViewModifications` is itself just a representation of the aforementioned checkboxes and their states, but its methods are what's important.

- **On source load:** check for `reference`s as normal; when `pointers` are built, register `replacement`s and `extension`s to `ViewModifications`.
- **On source unload:** check for `pointers`s as normal; when `pointers` are deleted, unregister `replacement`s and `extension`s from `ViewModifications`.
- **On checkbox toggle:** apply (or remove) a `disabled` property on the relevant `Pointer`s (this may require tree-traversal—for instance if C replaces B and B replaces A, but the user disables B's replacement of A).
- **On pointer-resolution during rendering:** look up the `Pointer` in `ViewModifications`; only follow the `Pointer` if it's enabled in `ViewModifications`.

All logic above would need to be updated to respect the `disabled` property, and appropriately interface with `ViewModifications` to prevent a desync.

### Errata management

This system can, in theory, be used to handle errata. Although a new `reference` `type` would be useful since it's meaningfully distinct from a `replacement`, it would work basically the same way. This would satisfy people who want to maintain pre-errata versions (c.f. the 5e MPMM debacle), and make Pf2ools unique in its data-browsing capabilities, except...

no

I really don't want to handle this. It'd require retaining a copy of _each version_ of every book, and _reversing_ errata changes in already converted books. I'd also have to contend with Paizo being famously _not good_ at communicating errata. All-in-all, more effort than I think I'm willing to spend at least 🤣

But it's a nice idea nonetheless. Maybe if we get off the ground with other volunteer converters we can stretch for it.

#### Remaster management

This is something that isn't absurd though!

The PC1 _alarm_ spell replaces the CRB _alarm_ spell. Even if they don't say it outright, it's pretty obvious that it does. No sensible user would want to see both the PC1 and CRB versions as two, distinct entities in the left-hand list.

My initial idea was to therefore treat PC1 stuff as a `replacement` of CRB stuff (where a direct analogue exists). This is a good place to start, but, to satisfy people who prefer legacy data, I can think of two routes:

- Introduce a new `reference` `type`: `remaster`. It functions identically to `replacement`, except it respects the setting switch. Easiest for data; extra code in the app.
- Implement `ViewModifications` as described [above](#toggle-overwritten-data) first, and add a convenience button to disable remaster `replacement`s. Since there are name-changes and the `replacement`s won't be limited to just PC1, this would likely require either a curated list of remaster-`replacement`s or some additional code to _actively search_ each loaded Paizo source for remaster `replacement`s. This would require the most work overall, but I think it's wise to merge with `ViewModifications` if we want to have it!

### Tangential systems

Some very last things to specify, related to this whole deal, to smooth over the bumps.

#### Source dependencies

Instead of constantly throwing the user pop-ups that we need a bridge built, we can help them build the bridge in advance. We allow that `source`s can have two additional properties: `requiredSourceIDs` and `optionalSourceIDs`.

If either of the properties are defined, show a pop-up containing the following:

- A list of sources that will also be loaded (as defined by `requiredSourceIDs`)
- A series of checkboxes for each source that _can_ be loaded (as per `optionalSourceIDs`).

A test script will ensure that, if any in-repo brew contains a `reference`, it includes its `target`'s source in either `requiredSourceIDs` or `optionalSourceIDs`. If possible, I'd like to extend that to `@tag`s targetting other homebrew too!

> [!IMPORTANT]
> The bar for a source to be _required_ should be very high. For instance, in 5e, Kobold Press' Expanding Codex makes _literally no sense at all_ if you don't also have Creature Codex. By comparison, even though Tome of Beasts Lairs relies on Tome of Beasts extensively, you could very plausibly not want the entire source loaded, and you can still read the adventure without requiring all the other statblocks.
>
> Another use-case for `requiredSourceIDs` is for people to make custom, _private_ homebrew files automatically load in-repo homebrew acceptable at their table. But, in general, its use in the repo should be sparse.

#### View-changing sources

A user should be notified if a homebrew source will change how statblocks look ahead of time, rather than letting them make errors because they didn't read each statblock's footer. If a source containing a `replacement` is loaded, a pop-up should display warning the user of this. The user can then either acknowledge it or cancel the load.

`extension`s don't need this unless there is no feasible way to highlight the changes in the statblock itself.
