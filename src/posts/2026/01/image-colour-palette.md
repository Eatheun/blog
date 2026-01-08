---
title: 'Creating a colour palette from an image'
date: 2026-01-07
tags:
- linux ricing
- personalisation
thumbnail: <img class='my-4 w-3/4 h-3/4 mr-4' src='/assets/img/posts/colours/ss1.png' alt='Cool colour palette 1' />
---

<!-- excerpt -->
## Brief

I desperately needed a colour palette but I'm terrible with colours. So I created a tool to find the 'best-fitting' colours from an image (desktop wallpaper).

I decided to create the tool in both Python and Rust and loosely compare the colour accuracy and speed from both implementations.

All the final code can be found <a target='_blank' href='https://github.com/Eatheun/img_kmeans/tree/main/'>here</a>.
## Here's the results!

{% for idx in range(1, 5) %}
    <a
        href='/assets/img/posts/colours/ss{{ idx }}.png'
        data-lightbox='colour-palette-ss'
    >
        <img
            class='my-4'
src='/assets/img/posts/colours/ss{{ idx }}.png'
            alt='Cool colour palette {{ idx }}'
        />
    </a>
{% endfor %}

## Iteration 1

Basically 👏 we're going to use a simple (naive) <a target='_blank' href='https://en.wikipedia.org/wiki/K-means_clustering'><i>k</i>-means clustering</a> algorithm and initially implement it in Python. If you got lost, the Python code is <a target='_blank' href='https://github.com/Eatheun/img_kmeans/tree/main/'>here</a>.

Ok, but there's a couple problems with the code.

The <a href='https://pillow.readthedocs.io/en/stable/reference/Image.html'>Image</a> module in <a href='https://pillow.readthedocs.io/en/stable/'>Pillow</a>, defines a method called `getcolors` but it's basically a counter data structure...

... so the implementation looks a little weird from standard.

``` py
# in run_kmeans_on_img
img = Image.open(img_fn)
img.thumbnail((SIZE, SIZE))  # SIZE = 200
cc = img.getcolors(SIZE ** 2)

points = [
    (Clr(colour), n)
    for n, colour in cc
    if isinstance(colour, tuple)
]

...

# in run_kmeans
for p, n in points:  # ah that's weird
    for _ in range(n):
        # find closest mean
        min_dist = float('Inf')
        min_mu_idx = 0
        for i, mu in enumerate(means):
            curr_dist = mu.dist_to(p)
            if curr_dist < min_dist:
                min_dist = curr_dist
                min_mu_idx = i

        # refit means
        means[min_mu_idx].add(p)

means.sort(key=lambda mu: int(mu.get_mean().to_hex(), 16))
return means
```

2. The output is ~~terrible and illegible~~ **bespoke and script-friendly**.
<div class='flex -mb-8'>
        <a
            href='/assets/img/posts/colours/sample.jpg'
            data-lightbox='sample-image'
            data-title='sample.jpg'
            class='no-underline'
        >
            <figure class='mr-4'>
                <img class='w-screen my-0' src='/assets/img/posts/colours/sample.jpg' alt='Sample image' />
                <figcaption>sample.jpg</figcaption>
            </figure>
        </a>
        <a
            href='/assets/img/posts/colours/sample_img_term_out.png'
            data-lightbox='sample-image'
            data-title='Output of colours from sample.jpg'
            class='no-underline'
        >
            <figure>
                <img class='w-screen my-0' src='/assets/img/posts/colours/sample_img_term_out.png' alt='Terminal output from sample image' />
                <figcaption>uhhh thanks I guess???</figcaption>
            </figure>
        </a>
</div>

### So how do you use this?

So apart from the fact that the output is just lines of hexadecimal numbers, there's another script `create_colour_palette.py` to create a more extensive colour palette. It's implemented with commandline arguments so you'll have to run it as follows...

<a href='/assets/img/posts/colours/better_term_out.png' data-lightbox='better_colour_palette' data-title='still hard to read...'>
    <img src='/assets/img/posts/colours/better_term_out.png' alt='Colour palette created from sample image' />
</a>

...AND you can change some of the source code to actually see the colours!

```diff-py
 for base, light, dark, punchy, negative in modulations:
-    base.print_hex()
-    light.print_hex()
-    dark.print_hex()
-    punchy.print_hex()
-    negative.print_hex()
-lightest_clr.print_hex()
-super_light_clr.print_hex()
-darkest_clr.print_hex()
-super_dark_clr.print_hex()
+    base.print_sample()
+    light.print_sample()
+    dark.print_sample()
+    punchy.print_sample()
+    negative.print_sample()
+lightest_clr.print_sample()
+super_light_clr.print_sample()
+darkest_clr.print_sample()
+super_dark_clr.print_sample()
```

<a href='/assets/img/posts/colours/coloured_term_out.png' data-lightbox='cool_colours' data-title='WHOA COLOURS!!!'>
    <img src='/assets/img/posts/colours/coloured_term_out.png' alt='Colour palette created from sample image' />
</a>

But I wouldn't recommend scripting with the colours on since they print ANSI escape codes. It's just a fun feature I implemented 🤩

So, anyways, you're probably interested in what these colours represent in the `create_colour_palette.py` script. The order matters here so listen up.

Firstly, the script sorts all the colours from the output of `img_kmeans.py` by 'vibrancy', which is my term for how grayscale (close to white, gray or black) a colour is. It will grab the 3 most vibrant colours.

```py
# find 3 most vibrant colours
vibrancies = list(map(lambda clr: (clr, clr.get_vibrancy()), clrs))
vibrancies.sort(key=lambda v: v[1], reverse=True)

modulations = []

for clr, vibrancy in vibrancies[:3]:  # over here we get the first 3
    ...
```

Then, it will modulate each colour into lighter and darker tones, as well as a 'punchier' or more vibrant tone and a negative.
```py
for clr, vibrancy in vibrancies[:3]:  # over here we get the first 3
    lighter = clr.lighter(mod)
    darker = clr.darker(mod)

    # for punchy, well add mod to some indices
    rgb = list(clr.get_rgba()[:-1])
    highest_idx, _, lowest_idx = tuple(map(
        lambda kv: kv[0],
        sorted(
            enumerate(rgb),
            key=lambda kv: kv[1],
            reverse=True
        )
    ))
    rgb[highest_idx] = min(rgb[highest_idx] + mod * 2, 0xff)
    rgb[lowest_idx] = min(rgb[lowest_idx] + mod, 0xff)
    punchy = Clr(tuple(rgb))

    negative = Clr(tuple(map(lambda val: 0xff - val, rgb)))

    # append to palette
    modulations.append((clr, lighter, darker, punchy, negative))
```

Now that we have the modulations, we also set some extremes for the colours by retrieving the lightest and darkest shades from the `img_kmeans.py` output and then modulating each of these into a super light and super dark shade respectively.

Finally, we print in the following order:
```py
# for each of the 3 most vibrant colours, print:
# - base
# - lighter
# - darker
# - punchy
# - negative
# print lightest and then super light
# print darkest and then super dark
for base, light, dark, punchy, negative in modulations:
    base.print_hex()
    light.print_hex()
    dark.print_hex()
    punchy.print_hex()
    negative.print_hex()
lightest_clr.print_hex()
super_light_clr.print_hex()
darkest_clr.print_hex()
super_dark_clr.print_hex()
```

Now you can pipe the output to some personal script and start personalising your desktop and apps!

## Iteration 2

Very simple modification to fix colour inaccuracy. The naive *k*-means algorithm is inherently flawed every few iterations so I implemented the <a target='_blank' href='https://en.wikipedia.org/wiki/K-means%2B%2B'><i>k</i>-means++</a> initialisation algorithm.
```py
# do k means ++ to find optimal clusters
means = []
for i in range(k):
    if len(means) == 0:
        first_mean = MeanClr(choice(points)[0].get_rgba())
        first_mean.compute_all_dists(points)
        means.append(first_mean)
        continue

    x = []
    min_l2_norms = []
    for p, _ in points:
        # compute all the l2_norms
        l2_norms = []
        for mu in means:
            dist = mu.dist_to(p)
            if dist == 0:
                continue

            l2_norms.append(dist ** 2)

        if len(l2_norms) == 0:
            continue

        # add
        x.append(p)
        min_l2_norms.append(min(l2_norms))

    total_l2_norms = sum(min_l2_norms)
    px = list(map(lambda d: d / total_l2_norms, min_l2_norms))
    new_mean = MeanClr(np.random.choice(x, p=px).get_rgba())

    means.append(new_mean)
```

That's pretty much it! If you're happy, go nab my implementation and go crazy with personalisation and scripting.

Here's a better look at some of the customisation I've done for my window manager (<a target="_blank" href="https://dwm.suckless.org">dwm</a>), text editor (<a target="_blank" href="http://neovim.io">Neovim</a>), browser (<a target="_blank" href="https://vivaldi.com">Vivaldi</a>) and terminal emulator (<a target="_blank" href="https://gnunn1.github.io/tilix-web/">Tilix</a>):

<a href='/assets/img/posts/colours/personalisation_wp.png' data-lightbox='personalisation'>
    <img src='/assets/img/posts/colours/personalisation_wp.png' alt='Window manager' />
</a>

<a href='/assets/img/posts/colours/personalisation_nvim.png' data-lightbox='personalisation'>
    <img src='/assets/img/posts/colours/personalisation_nvim.png' alt='Neovim' />
</a>

<a href='/assets/img/posts/colours/personalisation_term_brwsr.png' data-lightbox='personalisation'>
    <img src='/assets/img/posts/colours/personalisation_term_brwsr.png' alt='Tilix and Vivaldi' />
</a>

Also if you're interested in the Rust implementation, check out my code <a target="_blank" href="https://github.com/Eatheun/img_kmeans/tree/main/rust">here</a> 🙌
