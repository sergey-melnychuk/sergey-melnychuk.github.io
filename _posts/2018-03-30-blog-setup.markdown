---
layout: post
title:  "Blogging with GitHub Pages and Jekyll"
date:   2018-03-30 15:06:41 +0200
categories: blog jekyll github
---
Quite for some time I was thinking about starting a blog on software engineering and [GitHub Pages][github-pages] seemed like an ideal option, hosting the blog right next to the code. GitHub Pages comes with support of Jekyll static site generation engine, so using it seemed like a good option as well.

So today is the day! I'm starting the blog with the post describing the steps to set it up. [Jekyll docs][jekyll-docs] describe pretty much what needs to be done in order to get the blog up and running. Checking out [thoubleshooting][jekyll-setup] page is a good idea when it comes to sorting out permissions issues (I am not really comfortable with running blog-related commands with 'sudo' prefix). I have chosen simple and elegant theme [whiteglass][jekyll-theme], that comes with support of navigation, archive and about page. Updating links and footer content is simple by following the instructions provided with the theme.

Posts are created in [markdown][markdown-link] language, which I haven't used before, but it looks very simple. Here go some examples of what and how can be described in markdown.

Text decorations include **bold**, *italic*, ~~crossed out~~. And can be propagated _not only to words_ but to __whole sentences__.

Links can be included [in place!](http://google.com)

1. Numbered lists
1. are created with one-point prefix
1. and are turn into indices

- Bullet points
- are created with dashes
- before each list item
  - and can be intented
  - using two spaces
  - before the dash char
  - in sub-lists
  
Images are added in a way similar to links (source: [imgur](https://imgur.com/XhME3))
![Random image on the Internet](https://i.imgur.com/XhME3.jpg)

# (1) Headers
## (2) Are created
### (3) Using the hash signs
#### (4) Number of hashes matches
##### (5) Then N in \<hN> tag
###### (6) And can go from 1 to 6

Quotes are represented with greater signs as first char in the line of text:
> Imagination is more important than knowledge. For knowledge is limited to all we now know and understand, while imagination embraces the entire world, and all there ever will be to know and understand.
>
> Albert Einstein

> Quoting your own words feels awkward...
> 
> Sergey Melnychuk

Putting code on the page can be done inline like this `Cat cat = new Cat();` or as a syntax-highlighting enabled block:

```java
// Yes, this is hello world in Java,
// first what came to my mind when
// I was thinking about example for 
// a block of code
public class Main {
    public static void main(String args[]) {
        System.out.println("Hello, World!");
    }
}
```

Nice thing to have is a task list:
- [x] this is completed item
- [ ] this is not-completed item

Tables can be useful as well:

Color | Action
----- | ------
Red | Stop
Yellow | Think again
Green | Go

[github-pages]:  https://pages.github.com/
[jekyll-docs]:   https://jekyllrb.com/docs/home/
[jekyll-setup]:  https://jekyllrb.com/docs/troubleshooting/
[jekyll-theme]:  https://yous.be/whiteglass/about/
[markdown-link]: https://guides.github.com/features/mastering-markdown/
