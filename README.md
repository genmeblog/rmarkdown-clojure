Generating documents with `knitr` and Clojure
=============================================

This document is rendered by R `knitr` package with embedded Clojure code. Yes, it's possible. The renderer is configured to use `nRepl` client: [`rep`](https://github.com/eraserhd/rep).

What is knitr in short
----------------------

Knitr is R package which generates really variety documents out of markdown file with embedded code.

Let's run something
-------------------

First let's define `data`.

``` clojure
(def data {:a [1 2 3]
           :b [3 4 5]})
data
```

    #'user/data
    {:a [1 2 3], :b [3 4 5]}

Code was executed, `data` is defined and we can run another chunk.

``` clojure
(keys data)
```

    (:a :b)

And another one (everything is kept in `user` namespace).

``` clojure
(->> data
     vals
     (apply concat)
     (reduce +))
```

    18

Pretty print
------------

``` clojure
(clojure.pprint/pprint (repeatedly 6 #(repeatedly 3 rand)))
```

    ((0.295194802367057 0.8144531576951815 0.6147424047930444)
     (0.11617617843653338 0.3526533307892574 0.5204994714759178)
     (0.5233494270902467 0.30188286491945326 0.7827153625362437)
     (0.13981687786449426 0.3257476938039289 0.7656919090771052)
     (0.06439344277307846 0.4085124748701976 0.5709262295648099)
     (0.9918394380958168 0.3883005708118994 0.4213411020505128))

Generate image
--------------

``` clojure
(require '[clojure2d.core :refer [save]]
         '[clojure2d.color :as c]
         '[clojure2d.extra.utils :as u])
```

``` clojure
(-> :cubehelix
    (c/gradient)
    (u/gradient->image true)
    (save "gradient.png"))
```

![Generated gradient with luma](gradient.png)

Generate markdown
-----------------

``` clojure
(println "
test/data/stocks.csv [5 3]:

| symbol |       date | price |
|--------+------------+-------|
|   MSFT | 2000-01-01 | 39.81 |
|   MSFT | 2000-02-01 | 36.35 |
|   MSFT | 2000-03-01 | 43.22 |
|   MSFT | 2000-04-01 | 28.37 |
|   MSFT | 2000-05-01 | 25.45 |
")
```

test/data/stocks.csv \[5 3\]:

| symbol | date       | price |
|--------|------------|-------|
| MSFT   | 2000-01-01 | 39.81 |
| MSFT   | 2000-02-01 | 36.35 |
| MSFT   | 2000-03-01 | 43.22 |
| MSFT   | 2000-04-01 | 28.37 |
| MSFT   | 2000-05-01 | 25.45 |

How to setup
------------

I'm using Emacs with CIDER here.

-   Clojure
    -   Download and install [`rep`](https://github.com/eraserhd/rep)
    -   Be able to run `nRepl`
-   R
    -   Install R with `knitr` and `rmarkdown` packages (and all needed deps, like `pandoc`)
-   Emacs
    -   Install `ESS`, `poly-R` package which enables REPL inside Markdown file.

Run `nRepl`, create `.Rmd` file and add below chunk at the beginning of it. As you can see, there is a place to define `nrepl_port`. Find your port and change this value. I haven't been able to find an easy way to setup it automatically (yet).

```` markdown
```{r setup, include=FALSE}
find_nrepl_port_up <- function() {
    wd <- getwd()
    while(wd != dirname(wd)) {
        f <- paste0(wd,"/.nrepl-port")
        if(file.exists(f)) return(paste0("@",f))
        wd <- dirname(wd)
        f <- NULL
    }
}
port_file <- find_nrepl_port_up()
if(is.null(port_file)) stop("nREPL port not found")
library(knitr)
knitr_one_string <- knitr:::one_string
nrepl_cmd  <- "rep"
opts_chunk$set(comment=NA, highlight=TRUE)
knit_engines$set(clojure = function(options) {
    rep_params <- if((options$results == "asis") || isTRUE(options$stdout_only)) {
                      "--print 'out,1,%{out}' --print 'value,1,' -p"
                  } else {
                      "-p"
                  }
    code <- paste(rep_params, port_file, shQuote(knitr_one_string(options$code)))
    out <- if (options$eval) {
               if (options$message) message('running: ', nrepl_cmd, ' ', code)
               tryCatch(
                   system2(nrepl_cmd, code, stdout = TRUE, stderr = TRUE, env = options$engine.env),
                   error = function(e) {
                       if (!options$error) stop(e)
                       paste('Error in running command', nrepl_cmd)
                   }
               )
           } else ''
    if (!options$error && !is.null(attr(out, 'status'))) stop(knitr_one_string(out))
    engine_output(options, options$code, out)})
```
````

When it's done you can generate documents (html, pdf, whatever) within `ESS` or from external R session.

``` r
library(rmarkdown)
render("README.Rmd","all")
```

Emacs view
----------

![Emacs in action](emacs.png)

Rendered documents
------------------

-   [HTML](https://genmeblog.github.io/rmarkdown-clojure/README.html)
-   [PDF](https://github.com/genmeblog/rmarkdown-clojure/blob/master/README.pdf)
-   [WORD](https://github.com/genmeblog/rmarkdown-clojure/blob/master/README.docx)

What's odd
----------

There are couple of problems:

-   manual renderer setup
-   no pretty printing results by default

RMarkdown references
--------------------

-   <https://bookdown.org/yihui/rmarkdown/>
-   <https://bookdown.org/yihui/rmarkdown-cookbook/>
