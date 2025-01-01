Title: Lazy Images in Fulcro RAD
Date: 2024-12-31
Tags: fulcro, rad, semantic-ui, fomantic-ui, frontend, asked-clojure, clojure, frontend

> **Note:** This post is the first in a planned on-going series I'm calling
[asked-clojure](tags/asked-clojure.html). The idea is to turn questions I've
asked the Clojure community into blog posts, allowing me to 1) review what I've
learned and 2) provide more clarity on the subject for anyone that might find it
helpful.

In the [Slack discussion][Original discussion link], I asked how I could
achieve lazy image loading in a RAD report and linked to a portion of the
Fomantic-UI (a fork of Semantic-UI, which RAD uses for its rendering) docs on
visibility:

>  Visibility provides a set of callbacks for when a content appears in the
>  viewport

They also provide an [example][Fomantic Docs - Visibility] of using visibility specifically
for lazy loading images. The task is to try to adapt these components and
examples to a Fulcro code base.

I included a small snippet showing what I had tried already and Tony Kay soon
responded that he was unfamiliar with my approach and suggested an alternative
route using React's `componentDidMount` lifecycle method.

I continued on with the approach I was trying and found something that works, so
I decided to review and document my solution here.

Here are the namespace aliases and symbol references from below:

```clojure
[cljs.core :refer [js->clj]]
[com.fulcrologic.fulcro.components :as comp :refer [defsc]]
[com.fulcrologic.fulcro.dom :as dom :refer [div]]
[com.fulcrologic.fulcro.dom.html-entities :as ent]
[com.fulcrologic.rad.report :as report]
[com.fulcrologic.rad.report-options :as ro]
[com.fulcrologic.semantic-ui.factories :as sui]
```

## Objective: lazy load images in a RAD report

I'm using Fulcro [RAD][] for a number of projects, so it's always nice to learn
how to leverage RAD for more. In this case, I have a RAD report with an image
column and my objective is to load these images lazily as their parent row comes
into the viewport. In my original implementation, when the page was first
opened, all the ~100 images were being loaded right away, even if they were far
down the page and very much out of view. I wanted to have the page load with a
placeholder image used in this image column for all the rows that were not yet
in view. Upon scrolling down to put the row in view, the true image should be
loaded from the server to replace the placeholder image. The point of the
placeholder is that it only needs to be downloaded once.

## Method: use Semantic UI React factories

It turns out the Semantic-UI (aka SUI) library includes some helpers for this
sort of thing and the [fulcrologic/semantic-ui-wrapper][] library makes it easy
to use these in Fulcro apps by providing component factory functions. These
factories were auto-generated from the original Semantic-UI source to include
the original function headers and docstrings. While generating such library
wrappers will be the subject of future posts, for now I just wanted to mention
it here to highlight how flexible Fulcro is with regard to frontend rendering.

## Let's see some code already!
### A `LazyImage` Component

Here I use `defsc` to define a stateful React component factory which wraps the
auto-generated SUI factories and composes them together into a single thing.
Since we need to both display an image and dynamically control it depending on
its visibility, we have to combine both the `ui-image` and `ui-visibility`
factories. Further, we need to add a bit of state management (`loaded?`) so we
can use it in the `:onUpdate` handler.

```clojure
(defsc LazyImage
  "A component that lazy-loads images when they become visible in the viewport."
  [this {:keys [src placeholder className alt default-loaded?]}]
  {:initial-state (fn [_] {:loaded? false})}  ➊
  (let [{:keys [loaded?]} (comp/get-state this)]
    (sui/ui-visibility
      {:onUpdate
       (fn [_ event-data]
         (let [onScreen (-> (js->clj event-data :keywordize-keys true)  ➋
                            :calculations
                            :onScreen)]  ❸
           ;; If the image is visible but hasn't loaded yet, mark it as loaded
           (when (or (and onScreen (not loaded?))
                     default-loaded?)
             (comp/set-state! this {:loaded? true}))))
       ;; Make sure the callback fn fires immediately after mount
       :fireOnMount true}
      (sui/ui-image
        {:className (or className "ui image")
         :alt alt
         :src (if (or default-loaded? loaded?) src placeholder)}))))  ❹

;; Create a re-usable factory from the component
(def ui-lazy-image (comp/factory LazyImage))
```

    ❶ Assume images are not loaded until explicitly marked as such
    ➋ Comes in as a JS object that we convert for convenience
    ❸ :calculations from the update event that triggered the callback
    ❹ If the image is loaded, show it; otherwise, show placeholder image

### RAD Report: Column Formatter

In order to actually use `ui-lazy-image`, we need to find a way to include its
output in our DOM. For this particular report, I'm letting RAD do most of the
rendering, only adding some wrapper divs for consistent styling with the rest of
the site. It starts off looking like a standard RAD report:

```clojure
(report/defsc-report ItemList
  [this props]
  {;; report options map
   ;; ...
   ro/columns  [r.item/name
                r.item/location
                r.item/image-url]}
  (dom/div :.ui.container.homepage
    (dom/div :.row
      (dom/div :.sixteen.wide.column
        ent/nbsp
        ;; Let RAD handle the rest of the rendering.
        (report/render-layout this)))))
```

I've omitted most of the options map. Refer to the RAD docs for details. The
main thing to know is that there is _a column_ that receives an image URL
string.

The next step is to change the way this column is rendered. Of course, there are
multiple ways to achieve this, but I chose to keep it simple for now and keep it
all contained within the logic of this specific `ItemList` report. As such, the
first place I looked is in the [RAD report-options][] namespace, where I spotted
a fitting option called `column-formatters`. This option is described as taking
a map from a qualified key to a function that receives report and column details
and returns a string or element:

```clojure
ro/column-formatters
{:item/name (fn [report-instance value-to-format row-props attribute]
              ;; Replace with custom formatting
              string-or-element?)}
```

With this in mind, I went ahead with fleshing out what I want the column to look
like:

```clojure
(fn [_this v {:item/keys [location
                          image-url] :as props}]
  (let [row-idx (:com.fulcrologic.rad.report/idx (comp/get-computed props))  ➊
        max-rows 10
        default-loaded? (< row-idx max-rows)]  ❷
    (dom/h4 :.ui.image.header.profile-content
           ;; Use our lazy image factory!
           (ui-lazy-image {:src (str "/images/" image-url)  ❸
                           :placeholder "/images/square-image-placeholder.png"
                           :alt (str v " Profile Photo")
                           :className "mini rounded profile-image"
                           :default-loaded? default-loaded?})  ❹
           ;; Some additional wrapping for styling
           (div :.content v
             ;; Pulled location out of row props to style alongside name
             (div :.sub.header.profile-location location)))))
```


    ❶ Get the index of the row being rendered
    ➋ Decide whether the current row should always be loaded
    ❸ Use the `ui-lazy-image` factory from above with some params
    ❹ If `:default-loaded?` is true, the image should be loaded even when not in
      viewport

Upon initial page load, the top 10 rows load the actual image right away due to
the `max-rows` setting. This prevents a fresh page load from being populated
with placeholder images. As the user scrolls down, images are marked as loaded
and the placeholders are replaced with the real images as they come into view.

# Wrapping Up

All we did here was make use of the `visibility` behavior from Semantic-UI in
order to dynamically load images when their parent rows in the RAD report came
into view. The semantic-ui-wrapper library for Fulcro provides convenience
wrappers for most of the Semantic-UI stuff to make using it from Clojurescript
easy and idiomatic. There was a little bit of experimentation required in order
to get the result to feel right, but once we know about
`com.fulcrologic.semantic-ui.factories/ui-visibility`, that's just a matter of
tuning the arguments.

It's also worth noting that `ui-visibility` contains _many_ more optional
parameters that could further be used to adjust the exact behavior to your
preferences. This includes other callbacks like `onTopVisible`,
`onTopVisibleReverse`, `onPassingReverse` and many others, along with options
like `offset`.

More broadly, just about anything that the Semantic-UI library provides can be
easily used in a similar manner. Auto-generating convenience wrappers for other
Javascript libraries makes it easy to style your frontend however you like while
continuing to lean on Fulcro's full-stack data management to build your
applications.

[Original discussion link]: https://clojurians.slack.com/archives/C68M60S4F/p1731689329367159
[Fomantic Docs - Visibility]: https://fomantic-ui.com/behaviors/visibility.html#lazy-loading-images
[RAD]: https://book.fulcrologic.com/RAD.html
[fulcrologic/semantic-ui-wrapper]: https://github.com/fulcrologic/semantic-ui-wrapper/
[RAD report-options]: https://github.com/fulcrologic/fulcro-rad/blob/bdf1be102acb80576b63ea6c15e410a723e2b202/src/main/com/fulcrologic/rad/report_options.cljc
