+++
title = "Don't go looking for you first open source contribution"
date = 2020-10-18
description = "The traditional method of approaching one's first open source contribution gamifies the process and leads an unhelpful experience for all involved."
[taxonomies]
tags = ["open-source"]
+++

Contributing to open source software for the first time is a noble goal for new programmers, but it nevertheless proves daunting for many. Of course, there's [192 million projects on GitHub](https://github.com/search) to choose from, but where to start? That question has proven to be so elusive that seemingly an entire industry has been formed around it.

Searching "first open source contribution" yields hundreds of YouTube videos addressing the topic, along with sites like [firsttimersonly](https://www.firsttimersonly.com/) and [firstcontributions](https://firstcontributions.github.io/), which try to provide resources and suggest projects for those contributing for the first time. Many of these resources share essentially the same advice to programmers faced with the "first contribution" problem: go to a large code base like VSCode or React, pick a small bug to fix, send your pull request, and *voilà*, first contribution complete!

Take a look, for instance at firstcontributions, which has the headline "Make your first open source contribution in 5 minutes" and points to a list of large repositories. Firsttimersonly is similarly targeted at making large repositories more accessible to new contributors with a `good first issue` tag.

I support the intent behind these resources, and I don't mean for this article to disparage them, but I believe this advice can not only be frustrating for new programmers, but is **fundamentally unhelpful to all involved**. The problem with this solution is that big code bases like these are *big*. They have years of history behind them and are often backed by massive corporations employing a team of professional developers. Any issue that may be in reach for a new programmer, (one without experience dealing with code bases on this scale) would necessarily be so unimportant and isolated that fixing it would have virtually no impact on the project. 

This kind of effort is similarly unhelpful for contributor, since other than learning the basic work flow of open source development, they won't pick up any meaningful programming skills by fixing a trivial bug or documentation typo, nor will they feel like they've truly improved an open source project. Contributing in this way may tick a box, but it doesn't significantly benefit the project or the contributor.

## A better way

There's a different way to approach contributing to open source that may be less immediate, but can actually be helpful to both the contributor and the project. The method I will outline here is one that I used personally this summer to merge my first pull request into a real project, after becoming frustrated with the "first contribution" culture. Note that to veteran developers, nothing here will seem revolutionary, but consider how different this approach is to the advice usually given to those contributing for the first time.

### Use (small) open source software

The first step is easy. Find a few interesting open source projects from GitHub's 192 million and use them regularly. At this point, you don't even need to think about their code; just use the software consistently if it is user facing, or integrate it into your code if it is a library. Eventually, you will find a bug or a small feature improvement you could make to the project.

In my case, I contributed to the library [caire](https://github.com/esimov/caire), which implements content aware image resizing in Go, while using it in small side project I was working on. The code in the library is well documented and fairly small, with its main functions contained to a few files. At one point, it failed to resize a certain image, complaining that it couldn't preserve its dimensions while scaling. This struck me as odd, but also annoying -- my project involved resizing arbitrary images scraped from Reddit, so an unpredictable failure on some images was not ideal. 

Note the important difference between targeting this bug and a random one I could pick out from a large repository. This bug impacts something I'm working on, so I'm more invested in fixing it, and it lives in a small code base that can be understood at a high level within a day.

### Dive into the fix

The next step is to fork the code and start investigating the issue. For my case, the issue lied in how caire handled rescaling images before it applied content aware resizing.  In this part of the code, it scales the input proportionally before applying content aware resizing.

``` go
if p.NewWidth > p.NewHeight { // picks which side of the new image is larger
    newWidth = 0

    // rescales proportional to the larger side's length
    newImg = resize.Resize(uint(p.NewWidth), 0, img, resize.Lanczos3)
    
    if p.NewHeight < newImg.Bounds().Dy() {
        newHeight = newImg.Bounds().Dy() - p.NewHeight
    } else { // this error will be returned when original's aspect ratio is greater than target aspect ratio
        return nil, errors.New("cannot rescale to this size preserving the image aspect ratio")
    }
} else {
    newHeight = 0
    newImg = resize.Resize(0, uint(p.NewHeight), img, resize.Lanczos3)
    if p.NewWidth < newImg.Bounds().Dx() {
        newWidth = newImg.Bounds().Dx() - p.NewWidth
    } else {
        return nil, errors.New("cannot rescale to this size preserving the image aspect ratio")
    }
}
```

The problem is that the program always scales proportional to the larger side of the target aspect ratio. This only works when the aspect ratio of the original image is less than the target's, otherwise an error will be returned that the image can't be proportionally scaled. 

My change simplifies this step by scaling proportional to the smaller scale factor.

``` go
// calculate scale factors between original and target dimensions
wScaleFactor := float64(c.Width) / float64(p.NewWidth)
hScaleFactor := float64(c.Height) / float64(p.NewHeight)

// scale width and height are proportional to the smaller scale factor
scaleWidth := math.Round(float64(c.Width) / math.Min(wScaleFactor, hScaleFactor))
scaleHeight := math.Round(float64(c.Height) / math.Min(wScaleFactor, hScaleFactor))

// rescales to the calculated scale width and height
newImg = resize.Resize(uint(scaleWidth), uint(scaleHeight), img, resize.Lanczos3)

// The amount remaining to be removed by content aware resizing.
newWidth = int(scaleWidth) - p.NewWidth
newHeight = int(scaleHeight) - p.NewHeight
```

Now caire can resize any image to any target aspect ratio, rather than choking on some combinations of the two.

### Submit changes

The changes I made passed all tests, so they were merged in [this pull request](https://github.com/esimov/caire/pull/60). Even if they weren't merged, though, the changes I made were still useful to me, since they fixed a bug that actually impacted me, in contrast to a token issue you could find marked as a `good first issue` in a large repository.

## Wrapping up

By approaching open source contribution with a different mindset, we can make changes that are actually useful, and which can give you valuable experience working with a real code base. The main take away is summarized in this post's title: don't go looking for open source contributions, especially if you are an inexperienced developer. It will be easier, more satisfying, and more helpful to work on issues you encounter with small software that you actually use.

This method may not be as immediately satisfying; it's not something you can go do in the next ten minutes to complete your goal of contributing for the first time. But because it avoids the gamification of open source development, this approach will lead to more significant changes for the project, and more valuable experience for you as a programmer.











