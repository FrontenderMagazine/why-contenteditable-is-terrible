<section name="a7c7" class=" section-first" score="2.5">The first time I sat
across a table from Jacob
([@fat][1]), he asked bluntly, “How do you write a text editor?”

I drew a tree structure on the whiteboard, waved my hands, and said “This is
a shitty editing surface.” Then I drew a column of boxes with arrows pointing to
arrays, waved my hands some more, and said “This is a good editing surface.
”

Jacob raised an eyebrow.

This post is what I would have said instead, if I had a year to think about it
.

### **Why ContentEditable Is Terrible: A Mathematical Proof** 

[ContentEditable][2] is the native widget for editing rich text in a web
browser. It is…sad.

I’m going to try to prove to you, with some hand-wavey math, that the current
approach of ContentEditable is broken. This is not because I think math is a 
persuasive way to make this argument. It actually makes the argument more 
alienating.

But I do think that text editors lead to lots of fuzzy, ill-defined questions
like “What does What-You-See-Is-What-You-Get (WYSIWYG) even mean?” and “What 
happens when you select this text and hit Enter?” Axiomatic math is the best 
toolkit I know to take fuzzy, ill-defined questions and sharpen them.

So what does WYSIWYG mean? A good WYSIWYG editor should satisfy the following 3
axioms:</section><section name="b1b1" score="3.75">

1.  The *mapping *between DOM content and Visible content should be *well-
    behaved.
   * 
2.  The *mapping *between DOM selection and Visible selection should be *well-
    behaved.
   * 
3.  All visible edits should map *onto *an algebraically closed and complete
    set of visible content.
   </section><section name="95dc" score="2.5">

First, I’ll explain what each of these 3 axioms mean, and why a good editor
should obey these rules. But let’s be clear: they’re axioms. The weakest part of
any proof. We’re assuming they’re OK unless we have evidence otherwise.

Second, I’ll show that ContentEditable fails all 3 axioms.

Third, we’ll talk about how new browser features and libraries try to address
these issues, and how we handle them in the Medium editor.</section><section
name="920f" score="2.5
">

DOM space is the set of all web pages that you can express in HTML. All pages
can be represented as a tree of elements, with text nodes as the leaves of those
trees.

Visible space (“what-you-see-is-what-you-get”) is the set of all visible
pages — what you actually see on the screen when a browser renders a page. We 
say that two pages are the same in Visible space if they look exactly the same.

The browser’s rendering engine is a mapping from DOM space *onto *Visible space
. By “onto,” we mean that all Visible pages are the output of Render
(*x) *for some DOM tree *x.*

When we say a mapping is *well-behaved *in an editor, we mean that the mapping
preserves all edit operations (see footnote 1). More precisely, if*Render *is
well-defined, then

    for all edit operations E, and DOM pages x and y  
    Render(x) = Render(y)   
    implies Render(E(x)) = Render(E(y))

This is a way of formalizing the “what you get” part after “what you see
.” If two pages look the same, and we make the same edit on them, then the two 
results should look the same. (again, see 1
)

I’ve been surprised how many “WYSIWYG” editors on the web break this rule
. It may sound like an obvious principle. But it leads you into weird 
existential questions about what “same” means, which are best explored with 
examples.

### **Well-Behaved Content** 

Consider a sample sentence:

    The [hobbit][3] was a very well-to-do hobbit, and his name was ***Baggins.***

The Medium editor renders this sentence as, roughly, below.

    The <a href=”http://en.wikipedia.org/wiki/The_Hobbit">hobbit</a> was a very well-to-do hobbit, and his name was <strong><em>Baggins</em></strong>.

There are many, many ways to encode that last word, Baggins, as both italicized
and bold. (see footnote 2
)

    <strong><em>Baggins</em></strong>  
    <em><strong>Baggins</strong></em>  
    <em><strong>Bagg</strong><strong>ins</strong></em>  
    <em><strong>Bagg</strong></em><strong><em>ins</em></strong>

These forms should be equivalent, editor-wise. Any edits you make to this post
need to treat all these forms the same. It is surprisingly tricky to write an 
edit action that knows about all the different DOM forms.

For many ContentEditable implementations on the web, some invisible character
or empty span tag may slip into the HTML, so that two ContentEditable elements 
behave totally differently (even though they look the same). The experience can 
be maddening to users, and hard for engineers to debug.

Even if we knew how to write an edit action that is well-behaved, how would we
check it? If we limit our HTML to simple tags, proving that two forms are 
visually equivalent is…complicated. Your best bet is to iterate through each 
letter, assign it a style, and compare the results.

In an ideal world, we would have some high-level API for making “visual edits
” to the DOM. Each operation would guarantee that it is well-behaved, and does 
the “same” thing for all visually equivalent pages. Then, as long as your editor
only used these APIs, you could guarantee that it is well-behaved.</section><
section name="c67e" score="3.75
">

### **Well-Behaved Selections** 

The mapping between DOM content and Visible content is ugly, but at least it’
s many-to-one. One DOM representation has exactly one visible representation.

Selections are worse because the mapping is many-to-many.

It’s easy enough to see that one visible selection can have many DOM
representations. If you have the HTML,

    his name was <strong><em>Baggins</em></strong>

then a cursor before “Baggins” can be in one of three DOM positions: before
the*strong *start tag, between the *strong *start tag and the *em *start tag,
and after the*em *start tag. If you place your cursor before “Baggins” and
start typing, will your characters be bold, or italicized, or neither?<figure
name="776f" score="2.5
">

![][4]</figure>

More subtly, one DOM selection can have multiple visual representations.
Consider the case where “well-to-do” breaks after “to-”, as in the image above. 
A cursor at the end of the first line and at the beginning of the second line 
have the same DOM position, but different visual positions. As far as I know, 
there is no way to tell the browser to prefer one visual position over the other.

When designing editor commands, we want selections that look the same to behave
the same. But because that mapping is messy, this is a pain too.

### **Closed and Complete Edits** 

One day a couple years ago, my friend Julie sent me a Gchat message:

> We can remove Apple Style Span…Oh happy day!
Ryosuke Niwa wrote [a lovely post on the WebKit blog][5] about the quest to
remove apple-style-span. If you’ve read this far, many of the issues he raises 
will sound familiar. WebKit’s ContentEditable editor was adding loads of “
bookkeeping” HTML markup that didn’t change anything visually, but made the 
editor behave differently.

He also points out that WebKit’s ContentEditable implementation has to be
able to deal with HTML created by any other CMS, or any other browser’s 
ContentEditable implementation. Our editor should be a good citizen in this 
ecosystem. That means we ought to produce HTML that’s easy to read and 
understand. And on the flip side, we need to be aware that our editor has to 
deal with pasted content that can’t possibly be created in our editor.

I’ve seen classes of bugs where the only way to reproduce is to write text in
Firefox, switch to Chrome to make an edit, then switch back to Firefox. This is 
frustrating — for both developers and users.

To avoid this class of bugs, we say that the contents of a good WYSIWYG editor
should be algebraically closed under its edits. This means that the contents of 
the editor should always be something that I could create by typing “normally.” 
I shouldn’t be able to break out into different types of documents by pasting 
some HTML in, or editing in another browser.</section><section name="6fde"
score="3.75
">

### **A Framework For Good WYSIWYG Editors** 

A bare ContentEditable element is a bad WYSIWYG editor, because it breaks all
of these axioms. So how would we build a good WYSIWYG editor?

For the Medium editor, there are 4 key pieces.

1.  Create a model of the document, with a simple way to tell if two models are
    visually equivalent
   
2.  Create a mapping between the DOM and our model
3.  Define well-behaved edit operations on this model
4.  Translating all key presses and mouse clicks into sequence of these
    operations
   

I’ll walk you briefly through each of these pieces, and how we make changes
to them. At the end, I’ll discuss how browser engineers are making 
ContentEditable better, and may make some of these components obsolete.</
section
><section name="f656" score="3.75">

### **The Medium Editor Model** 

The Medium editor model has two fields: a list of paragraphs, and a list of
sections.

Each paragraph contains

*   text, a string of plain text
*   markups, a list of formatting text ranges, like “bold from char 1 to 5”
*   metadata for images or embeds
*   layout, a description of how we should position the paragraph

A section describes a background for a sublist of paragraphs.

Any selection in the Medium editor is expressed as two points. Each point is a
paragraph index and text offset into that paragraph, and a type. Most selections
are text-type selections. We also have media-type selections (when the tooltip 
is on the image), and section-type selections (when the tooltip is on the 
section background
).

The advantage of this model is that two models have the same visual rendering
if and only if the models are equal. Any change to a model translates to a well-
defined visual change.</section><section name="f0dd" score="2.5">

### **The Medium Editor Mapping** 

Next, we define a mapping from DOM space to the model space. We break this into
two separate cases: “indoor” mappings and “outdoor” mappings.

An indoor mapping is when we take content inside the editor and translate it
back and forth between DOM and model. We expect an indoor mapping to be one-to-
one.

An outdoor mapping is when we have HTML from outside the editor, like when the
user pastes HTML from Word into a Medium post. We need to translate it to our 
paragraphs-and-sections model. We expect outdoor mappings to be lossy. We 
prioritize plain text first, then bold/italic/link markup, then images and other
miscellaneous formatting.

When we map our model to the DOM, the tree looks like this:

    <div> <!-- root -->  
    <section> <!-- section -->  
    <!-- section-inner -->  
    <div class="section-inner layout-column">  
    <p>  <!-- paragraph -->  
    <strong><em>Baggins</em></strong> <!-- text -->

The section node is generated from the section model, and applies background
images or colors to a list of paragraphs.

The section-inner node is generated from the paragraph’s layout property, and
determines the width of the main column. On most paragraphs, it’s narrow and 
centered. On full-width image paragraphs, it’s 100%. On row grids, it’s half-way
outset.

The next node is the semantic type of the paragraph: P, H2, H3, PRE, FIGURE,
BLOCKQUOTE, OL-LI (ordered list item), and UL-LI (unordered list item
).

When we translate markup ranges into DOM nodes, we sort them by type: A, then
STRONG, then EM. We will never print a STRONG tag containing an anchor. We break
it up so that the anchor contains the STRONG tag.</section><section name="3010
" score="2.5
">

### **Medium Edit Operations** 

The Medium body editor has exactly 6 edit operations: InsertParagraph,
RemoveParagraph, UpdateParagraph, InsertSection, RemoveSection, and 
UpdateSection.

They do just what they sound like. The paragraph operations take a paragraph
model and an index. The section operations take a section model and an index.

All possible editor contents can be expressed by a sequence of these operations
, and it’s usually trivial to construct such a sequence.

It’s easy to see that the content is well-behaved under these edit operations
. They act directly on our model, not on the DOM, and the model makes it easy to
tell when two things are visually equivalent.</section><section name="07f1"
score="2.5
">

### **Capturing Edits** 

When you interact with the Medium editor, we have to translate your key presses
and mouse clicks into a sequence of those 6 operations.

This is the trickiest part. We don’t want to keep around some huge list of
every possible key sequence. It would be a crazy long list for English-speaking 
users, never mind for languages with non-Latin characters and keyboards.

The key insight is that we *can *enumerate all the ways to insert and remove
paragraphs with normal ContentEditable keyboard commands. They are: carriage 
return (enter, ctrl-m, etc.), delete (delete, backspace, etc.), type-over (
select-text-and-type), and paste. So we capture, cancel, and manually translate 
those keyboard events into our internal editor operations.

For all other keyboard events, we let the native ContentEditable behavior kick
in. After the keyboard event finishes, we map the paragraph DOM back to a 
paragraph model, and compare the model to what we had before. If the DOM changed,
we create a new UpdateParagraph op and flush it through the editor pipeline, 
bringing the DOM and model back in sync.</section><section name="fa90" score="2
.5
">

### **Capturing Edits Quickly** 

If we had infinite computing power, applying these edit operations would be
straightforward. We would apply them to the model, re-render the whole post, and
be done with it.

But in the real world, re-rendering the whole post on every keypress would be
too slow. And you would see lots of ugly flickering, because iframes and images 
would be continuously reloading. Instead, we listen on changes to the model, and
try to make the minimal possible change to the DOM.

As I type this now, I can see the Chrome spellchecker’s red-underline on the
word “keypress” flicker. That’s because the Medium editor is changing the whole 
paragraph at once, rather than only changing a piece of the paragraph. If we 
made a more narrow DOM change, the flicker would go away, but the code would be 
more complicated.</section><section name="03f6" score="3.75">

### **Towards A Brighter Text Editing Future** 

There have been some rumblings lately from some Chromium contributors (Levi
Weintraub, Julie Parent, and Jelte Liebrand) that they want to redo 
ContentEditable on top of Polymer Elements and Shadow DOM. The proposal wrestles
with many of the same high-level architecture issues that the Medium editor 
tries to solve.

1.  Create an editor model made out of custom [Polymer elements][6] 
2.  Define a mapping between the editor model and the real DOM with a 
    [Shadow DOM][7] 
3.  All keypresses and mouse clicks in ContentEditable would be translated into
    an abstract edit intent, expressed as a JSON object like {editIntent: ‘delete
    ’}
4.  Polymer elements could define handers for edit intents

If the Medium editor got some sort of edit intent API, we would be able to
throw away a lot of custom code for translating keypresses into abstract edit 
operations. It would be an interesting experiment to express our paragraph 
models as Polymer/ShadowDOM elements.</section><section name="21f9" score="2.5
">

### **What ContentEditable Could Be** 

Whenever I explain this to people who work on text editors, they call me on my
sleight-of-hand here.

“Of course the Medium editor is better than ContentEditable. You cheated.
ContentEditable tries to be a general-purpose WYSIWYG HTML editor. The Medium 
editor drops the ‘general-purpose’ requirement, so you can pick and choose what 
HTML structures you want to handle.
”

This is true. But it’s misguided.

A good WYSIWYG editor is axiomatically inconsistent with a good general-purpose
HTML editor. It’s impossible to build what ContentEditable wants to be, because 
they have conflicting requirements.

My thinking on this is influenced a lot by Steve Yegge’s “
[The Nonesuch Beast][8]” essay. Design and UX problems can be just as
intractable as big-O algorithmic problems. A good WYSIWYG editor of arbitrary 
HTML is just as impossible as the halting problem is impossible.

ContentEditable can be salvaged. But its mission must change. With richer DOM
APIs like Shadow DOM, ContentEditable could become a platform for building a new
generation of editors on the web. But we have to treat it as an editor platform 
and API, rather than as a standalone component that tries to do everything 
itself.</section><section name="ff14" class=" section-last" score="3.75">

### **Footnotes** 

1.  If you’ve taken higher-level undergraduate math, a well-behaved mapping
    is a
   [morphism][9]. That word feels clunky, and carries additional baggage that
    doesn’t help us here. So we use
   *well-behaved *in this post.
2.  CSS complicates this discussion, and complicates the editing algorithms
    that browser engineers have to implement. But it doesn’t fundamentally change 
    the proof of why ContentEditable is terrible. Without loss of generality, we can
    ignore CSS, and restrict our analysis to simple HTML.
   </section>

 [1]: https://twitter.com/@fat
 [2]: https://dvcs.w3.org/hg/editing/raw-file/tip/editing.html
 [3]: http://en.wikipedia.org/wiki/The_Hobbit
 [4]: img/1*RNm0jdoSbUfRnkTgOQl9Sw.png
 [5]: https://www.webkit.org/blog/1737/apple-style-span-is-gone/
 [6]: http://www.polymer-project.org/
 [7]: http://glazkov.com/2011/01/14/what-the-heck-is-shadow-dom/
 [8]: https://sites.google.com/site/steveyegge2/nonesuch-beast
 [9]: http://en.wikipedia.org/wiki/Morphism