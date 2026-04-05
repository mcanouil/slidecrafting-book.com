# Slidecrafting

Making beautiful slides with reveal.js and Quarto

Author

Emil Hvitfeldt

Published

September 24, 2025

# Preface

Hello! This book is about what I like to call **slidecrafting**; The art of putting together slides that are functional and aesthetically pleasing. I will be using [quarto presentations](https://quarto.org/) throughout the whole book. Some of the advice will transcend Quarto presentations and apply to all kinds of slide technologies, but the book is written with Quarto in mind. There will thus inherently be some overlap with quarto documentation. It is suggested to read all the [quarto revealjs](https://quarto.org/docs/presentations/revealjs/index.html) documentation alongside this book for optimal learning.

I think of slidecrafting as an art because of the inexact nature of many of the decisions you will be making; after all, many slide decks can be distilled down into a series of still images, each of which should be carefully crafted.

This book will not be able to teach you everything you need, but it will cover the parts of the process that stay constant from deck to deck, so you can focus on the content and develop your brand/style.

The book will be split into a number of semi-coherent sections, each with a number of chapters. This is not a perfect subdivision, but it serves its purpose quite well.

## Theming

The first section is the **theming** section, where we go over how to change the visual appearance of your slides. It starts with a quick chapter showing you how to set up a style in less than [10 minutes](10-minute.llms.md). Then we will go into more details on how to use [colors](colors.llms.md), [fonts](fonts.llms.md), and [sizes](sizes.llms.md) to change the look and feel of your slides. The [theme](theme.llms.md) chapter goes over the process of developing a slide-by-slide theme. Lastly, we have a chapter going over some [SCSS tips and tricks](scss.llms.md), which I find generally useful but doesn’t fit into any of the other chapters.

## Content

The **contents** section includes a chapter on how to change what is on the slides, either for variety or different ways to highlight new things. The [elements](elements.llms.md) chapter focuses on things that show up on the slides, such as images and figures. The [layout](layout.llms.md) talks about different ways to move elements of the page around “geographically”. Sometimes you want to highlight very specific things in some code, and for that to work, you will likely need to [style the code manually](manual-code.llms.md). There are other times where you want to showcase code and its output, and that can be done using [asciicast](asciicast.llms.md).

## Interactivity

This section looks at how we [fragments](fragments.llms.md) to change things up even more, allowing for elements to change and more.

## Extensions

This section will go over a handful of extensions that you can use to spruce up your slides.

## miscellaneous

Lastly, there is the [miscellaneous](miscellaneous.llms.md) chapter, which will contain everything that doesn’t fit anywhere else.
