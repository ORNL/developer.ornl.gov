# ORNL Developer Blog

This blog is a place for ORNL developers to publish short
technical articles related to software development and use
activities at the lab.

It *is not* a replacement for scholarly articles, which should
be considered when you are publishing performance data,
surveys, or reviews.  It *is* a place for HOWTO-s and recipes
that are too short for a traditional journal.


# Process for contributing an article

Before writing up a complete article, create a [new issue](https://code.ornl.gov/software-sustainability/developer.ornl.gov/-/issues/new) briefly describing the title, tags, and abstract of the article you'd like to write.  The editorial should follow up on your issue with comments.

After deciding to go forward, create the proposed article in a file named like `_posts/YYYY-MM-DD-article-name.md`.  Commit the article to a new branch and send it to this repository as a merge request (aka pull request).


# Writing in markdown

There are four good ways to preview your articles
in markdown before committing them to the repository.

Content on this blog is displayed using [kramdown](https://kramdown.gettalong.org/) and [Liquid with jekyll](https://jekyllrb.com/docs/liquid/) to substitute inside braces, `{{ }} / {% %}`.

There are several methods to get a `built preview` of your
document as you're writing it.

### Method 1: direct issue

1. Open a [new issue](https://code.ornl.gov/software-sustainability/developer.ornl.gov/-/issues/new)

2. Select "article" from the Description drop-down to get a template.

3. Type your article directly into it, and the preview tab to see how the formatting looks.

Note: this will not substitute templates or put in fancy headers.


### Method 2: grip

1. Install the [grip](https://github.com/joeyespo/grip/) package for python using `pip install grip`

2. Draft your post in a file named `YYYY-MM-DD-article-name.md` while running `grip YYYY-MM-DD-article-name.md` and viewing in your browser.

Note: this will not substitute templates or put in fancy headers.


### Method 3: pandoc

1. Install [pandoc](https://pandoc.org/installing.html).

2. Create a preview by formatting to html or pdf and opening the result:

    pandoc filename.md -o filename.html

Note: this will not substitute templates or put in fancy headers.


### Method 4: serve this site locally

This method will create a full site preview, including substitution for liquid templates and includes, etc.  However, there is a bit more installation work.

1. Install [Ruby Version Manager](https://rvm.io/rvm/install)

2. Load rvm into your shell.  My preferred method for this is to use a function from my `.bash_profile`:

    ruby_env() {
      source $HOME/.rvm/scripts/rvm
      rvm use 3.0.0
    }

3. Clone and install this project:

    git clone git@code.ornl.gov:software-sustainability/developer.ornl.gov.git
    git checkout -b new-article
    bundle
    
4. Run the webserver and browse locally:

    bundle exec jekyll serve

5. Edit or create any content file, and refresh the browser.

