# Yes, that's the tracked branch



## Credits

* Github Page built on [Jekyll](http://jekyllrb.com/) 
* Theme by [Hydeout](https://github.com/fongandrew/hydeout) (theme's [README](README-hydeout.md)) 
* Archive page inspired from [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme
* Alerts (Tip/Warning/Info/...) inspired from [Documentation](https://github.com/tomjoht/documentation-theme-jekyll/) theme 



## Reminders

* Run locally with

  ```bash
  $ rbenv install 3.1.2 # this version prevents errors from some gems
  $ rbenv global 3.1.2
  
  $ bundle exec jekyll serve --livereload
  # after specifying new plugin in Gemfile
  $ bundle install
  ```

* Check build status on [Actions](https://github.com/LAripping/laripping.github.io/actions/workflows/pages/pages-build-deployment)

* Check how Shared links render in

  * Twitter's [Card Validator](https://cards-dev.twitter.com/validator) 
  * LinkedIn's [Post Inspector](https://www.linkedin.com/post-inspector/)
  * Facebook's [Sharing Debugger](https://developers.facebook.com/tools/debug/)

  for all types 

  1. [Home](https://laripping.github.io/) 
  2. A page such as [About Me](https://laripping.github.io/about.html) / Tags
  3. An article such as [the OSEP one](https://laripping.github.io/blog-posts/2021/04/29/noobs-guide-to-osep.html)



## TODOs

- [x] Re-refactor "related" - 2 common Tags and factor in categories 
- [x] Add everything
- [x] Fix Broken theme on narrow screens
- [ ] Add Projects page, sidebar-linked
- [x] Images on posts? Only on Index layout.
  - [x] Make them appear in social post preview (SEO)
- [x] Excerpt provided in MD directive. The rest only visible on post.html -> to unlock TOCs 
  - [x] add Advisory tables (Product+Version - Severity - CVE - type)    
  - [x] Re-add Intro/Introduction sections
- [ ] (sticky) TOC on the right side? ( [idea](https://github.com/mmistakes/minimal-mistakes/blob/master/_layouts/single.html) )
