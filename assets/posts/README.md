Images used inside blog posts go here.

Write the picture and the passage it should sit beside as ONE paragraph, with
the side class on the line above:

    {: .fig-left}
    ![Kanha frees the calf](/assets/posts/kanha-calf.webp)
    One day, as he passed by a household, he saw a calf which was tied to a
    post, trying frantically to free itself...

No blank line between the picture and the text — they have to be the same
paragraph, or the passage below can end up starting with a single stray word.
Alternate `.fig-left` and `.fig-right` down the post so the pictures zig-zag.
Below 600px wide they stack full-width automatically.

Two things to watch:

- Filenames must not begin with an underscore — Jekyll ignores those, so the
  image would 404 on the live site.
- Save as `.webp` and about 1000px wide. Each picture renders ~440px, so a
  3MB PNG downloads roughly ten times more than it shows.

If a passage is short and its picture hangs below the last line, add
`.fig-sm` beside the side class to make the picture narrower — it gets
shorter and the text column narrower, so the two meet:

    {: .fig-right .fig-sm}

(Centring the text against the picture is not possible here: the picture
lives inside the paragraph, so a flex row would break the passage into a
separate column at every *italic*.)

If a picture should have a *run* of short paragraphs beside it (dialogue,
say) rather than one passage, add `.fig-flow` and leave the picture in a
block of its own — everything after it flows around until the space runs
out:

    {: .fig-right .fig-flow}
    ![Kanha caught clinging to the butter pot](/assets/posts/fun.webp)

    **Kanha**: "Welcome home, mother!"

    **Latha**: "Can you please explain yourself?"

This is the one case that wants a blank line after the picture. It does not
contain its float, so leave a few paragraphs before the next picture.
