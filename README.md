[![Build Status](https://travis-ci.org/capitaomorte/snooze.svg?branch=master)](https://travis-ci.org/capitaomorte/snooze)
Snooze
=======

_Snooze_ is a framework for building REST web services in Common
Lisp.

The primary goal is to make the programmer stay as close to the
regular Lisp idioms as possible, even while writing with HTTP in mind.

Here's one such REST service to access Lisp documentation over HTTP:

```lisp
(defpackage #:readme-demo (:use #:cl #:snooze))
(in-package #:readme-demo)

(defun find-symbol-or-lose (name package)
  (or (find-symbol (string name) (find-package package))
      (http-condition 404 "Sorry, no such symbol")))

(defroute lispdoc (:get :text/* name &key (package :cl) (doctype 'function))
  (or (documentation (find-symbol-or-lose name package) doctype)
      (http-condition 404 "Sorry, ~a doesn't have any ~a doc" name doctype)))

(defroute lispdoc (:post :text/plain name &key (package :cl) (doctype 'function))
  (setf (documentation (find-symbol-or-lose name package) doctype)
        (payload-as-string)))

;; Let's use clack as a server backend
(clack:clackup (snooze:make-clack-app) :port 9003)
```

No regular expressions, annotations or funny argument-matching syntax: routes
not only **look like** functions, they **are** functions.

Here are the routes thus defined and some of the error reporting you
get for free:

```HTTP
GET /lispdoc/defun                         => 200 OK
GET /lispdoc/funny-syntax?package=snooze   => 404 Not found
GET /lispdoc/in/?valid=args                => 400 Bad Request
                                           
GET /lispdoc/defun                         => 406 Not Acceptable 
Accept: application/json

POST /lispdoc/scan?package=cl-ppcre        => 200 OK 
Content-type: text/plain

POST /lispdoc/defun                        => 415 Unsupported Media Type 
Content-type: application/json
```

Checkout the [tutorial](#tutorial), which builds on this
application. If you're intrigued about how and why URI paths become
symbols arguments to your routes, [read here](#how-snooze-converts-uri-components-to-arguments)

Ah, snooze is kinda **BETA** and the usual disclaimer of warranty applies.

Rationale
---------

_Snooze_ maps [REST/HTTP](https://en.wikipedia.org/wiki/REST) concepts
to Common Lisp concepts:

| HTTP concept                         | Snooze CL concept                   |
| :----------------------------------- | ----------------------------------: |
| Verbs (`GET`, `POST`, `DELETE`, etc) | CLOS specializer on first argument  |
| `Accept:` and `Content-Type:`        | CLOS specializer on second argument |
| URI path (`/path1/path2/path3)`)     | Required and optional arguments     |
| URL queries (`?param=value&p2=v2`)   | Keyword arguments                   |
| Status codes (`404`, `500`, etc)     | CL conditions                       |

This has some advantages, for example

* since in _Snooze_ every route is a method, you can `trace` it like a
  regular function, find its definition with `M-.` or maybe use
  `:around` qualifiers.
* the lambda-list format guarantees that URI errors can be spotted early.
* There is no need to write code to "extract" arguments from the
  URI. An URI generated by _Snooze_ for one of its routes always matches it.

There are many Common Lisp systems that provide HTTP routing like
[caveman][caveman], [cl-rest-server][cl-rest-server],[restas][restas]
and [ningle][ningle]. Unfortunately, they tend to make you learn some
extra route-definition syntax.


Tutorial
--------

Consider the sample [above](#snooze). Let's pick up where we left off,
and build a bit on our docstring-manipulating application. We'll see
how to:

* [dispatch on HTTP methods and content-types](#content-types)
* [generate and encode compatible URIs](#uri-generation)
* [grafully handle failure conditions](#controlling-errors)
* [control conversion of URI arguments](#how-snooze-converts-uri-components-to-arguments)
* [refactor routes without changing the API](#tighter-routes)
* [hook _Snooze_ into the backend of your choice](#other-backends)

This tutorial assumes you're using a recent version of
[quicklisp][quicklisp] so start by entering this into your REPL.

```lisp
(push "path/to/snoozes/parent/dir" quicklisp:*local-project-directories*)
(ql:quickload :snooze)
```

### Content-Types

Let's start by serving docstrings in HTML. As seen above, we already
have a route which matches any text:

```
(defroute lispdoc (:get :text/* name &key (package :cl) (type 'function))
  (or (documentation (find-symbol-or-lose name package) type)
      (http-condition 404 "Sorry no ~a doc for ~a" type name)))
```

To add HTML support, we just notice that `text/html` *is*
`text/*`. Also because routes are really only CLOS methods, the
easiest way is:

```lisp
(defroute lispdoc :around (:get :text/html name &key &allow-other-keys)
  (format nil "<h1>Docstring for ~a</h1><p>~a</p>"
          name (call-next-method)))
```

Though you should probably escape the HTML with something like
[cl-who][cl-who]'s `escape-string-all``. You might also consider
specializing directly to `text/html` once you start needing some more
HTML-specific behaviour.

Finally, let's accept `POST` requests with JSON content. In this
version we accept the `package` and `doctype` parameters in the JSON
request's body.

```lisp
(defroute lispdoc (:post "application/json" name &key (package :cl) (doctype 'function))
  (let* ((json (handler-case
                   ;; you'll need to quickload :cl-json
                   (json:decode-json-from-string
                    (payload-as-string))
                 (error (e)
                   (http-condition 400 "Malformed JSON (~a)!" e))))
         (sym (find-symbol-or-lose name package))
         (docstring (cdr (assoc :docstring json))))
    (if (and sym docstring doctype)
        (setf (documentation sym doctype) docstring)
        (http-condition 400 "JSON missing some properties"))))
```

### URI generation

_Snooze_ can generate the URIs for a resource. To get a function that
does that do:

```lisp
(defgenpath lispdoc lispdoc-path)
```

The generated function has an arglist matching your route's arguments:

```lisp
(lispdoc-path 'defroute :package 'snooze)
  ;; => "/lispdoc/defroute?package=snooze"
(lispdoc-path 'defun)
  ;; => "/lispdoc/defun"
(lispdoc-path '*standard-output* :doctype 'variable)
  ;; => "/lispdoc/%2Astandard-output%2A?doctype=variable"
(lispdoc-path '*standard-output* :FOO 'hey)
  ;; error! unknown &KEY argument: :FOO
```

Notice the automatic URI-encoding of the `*` character and how the
function errors on invalid keyword arguments that would produce an
invalid route.

Path generators are useful, for example, when write HTML links to your
resources. In our example, let's use it to guide the user to the
correct URL when a 404 happens:

```lisp
(defun doc-not-found-message (name package doctype)
  (let* ((othertype (if (eq doctype 'function) 'variable 'function))
         (otherdoc (documentation (find-symbol-or-lose name package) othertype)))
    (with-output-to-string (s)
      (format s "There is no ~a doc for ~a." doctype name)
      (when otherdoc
        (format s "<p>But try <a href=~a>here</a></p>"
                (lispdoc-path name :package package :doctype othertype))))))

(defroute lispdoc (:get :text/html name &key (package :cl) (doctype 'function))
  (or (documentation (find-symbol-or-lose name package) doctype)
      (http-condition 404 (doc-not-found-message name package doctype))))
```

If you now point your browser to:

```
http://localhost:9003/lispdoc/%2Astandard-output%2A?doctype=variable
```

You should see a nicer 404 error message. Except you don't, because by
default _Snooze_ is very terse with error messages and we haven't told it
not to be. The next sections explains how to change that.

Controlling errors
------------------

Errors and unexpected situations are part of normal HTTP life. Many
websites and REST services not only return an HTTP status code, but
also serve information about the conditions that lead to an error, be
it in a pretty HTML error page or a JSON object describing the
problem.

Snooze tries to make it possible to precisely control what information
gets sent to the client. It uses a generic function and two variables:

* `explain-condition (condition resource content-type)`
* `*catch-errors*`
* `*catch-http-conditions*`

Out of the box, there are no methods on `explain-condition` and the
first two variables are set to `t` by default.

This means that any HTTP condition or a Lisp error in your application
will generate a very terse reply in plain-text containing only the
status code and the standard reason phrase.

You can amend this selectively by writing an `explain-condition`
methods explain HTTP conditions politely in HTML:

```lisp
(defmethod explain-condition ((condition http-condition)
                              (resource (eql #'lispdoc))
                              (ct snooze-types:text/html))
               (with-output-to-string (s)
                 (format s "<h1>Terribly sorry</h1><p>You might have made a mistake, I'm afraid</p>")
                 (format s "<p>~a</p>" condition)))
```

You can use the same technique to explain *any* error like so:

```lisp
(defmethod explain-condition ((error error) (resource (eql #'lispdoc)) (ct snooze-types:text/html))
               (with-output-to-string (s)
                 (format s "<h1>Oh dear</h1><p>It seems I've messed up somehow</p>")))
```

Finally, you can play around with `*catch-errors*` and
`*catch-http-conditions` (see their docstrings). I normally leave
`*catch-http-conditions*` set to `t` and `*catch-errors*` set to
either `:verbose` or `nil` depending on whether I want to do debugging
in the browser or in Emacs.

How Snooze converts URI components to arguments
-----------------------------------------------

You might have noticed already that the CLOS generic functions that
represent resources take as arguments actual Lisp symbols extracted
from the URI, whereas other frameworks normally pass them as strings.

Let's drift from the `lispdoc` example a bit. Consider this app fragment:


```lisp
(defclass beatle () ((id      :initarg :id)
                     (name    :initarg :name    :accessor name)
                     (guitars :initarg :guitars :accessor number-of-guitars)))

(defvar *beatles* ...)

(defroute beatles (:get "text/plain" &key (key 'number-of-guitars) (predicate '>))
  (assert-safe-functions key predicate)
  (format nil "~{~a~^~%~}"
          (mapcar #'name
                  (sort (copy-list *beatles*) predicate :key key))))

(defgenpath beatles beatles-path)
```

`beatles-path`, when passed actual symbols naming functions that are
have meaning in your application domain, will generate perfect URI's
for accesing the `beatles` route. So

```lisp
(beatles-path :key 'name :predicate 'string-lessp)
```

returns

```lisp
"/beatles/?key=name&predicate=string-lessp"
```

Secondly this is only the default behaviour of Snooze, and is entirely
configurable: if you really want to have the path `foo/bar/baz` become
the arguments `"foo"`, `"bar"` and `"baz"` to your application you
merely need to add a CLOS method to the each of the generic functions
`read-for-resource` and `write-for-resource`.

I recommend you keep the default:

* `read-for-resource` uses a very locked down version of
   `cl:read-to-string`, one that doesn't intern symbols, allow any kind
   of reader macros or read anything more complicated than a number, a
   string or a symbol. The non-interning is for security.

* `write-for-resource` does basically the inverse of
  `read-for-resource`, writing onto a string representation that is
  read back equivalently.

If this is not enough, _Snooze_ allows even finer control over the way
the URI is translated, so that individual arguments are read
differently, or even combined. One might want, for example:

```
GET beatle/3
```

where `3` is an id, to pass an actual `beatle` object to the
route. This is explained in the next section.`

Tighter routes
---------------

The routes we have until now all use:

* the `find-symbol-or-lose` helper;
* the `:package` keyword arg.

It would be nicer if they were simply functions of a symbol: After all
, in Common Lisp, passing symbols around doesn't force you to pass
their packages separately!

So basically, we want to write routes like this:

```lisp
(defroute lispdoc (:get :text/* (sym symbol) &key (doctype 'function))
  (or (documentation sym doctype)
      (http-condition 404 (doc-not-found-message sym doctype))))
```

Actually, this will work *just fine* out of the box. But the routes
matched are not as human-readable like before: they look like this:

```
(lispdoc-path 'cl-ppcre:scan)
  ;; => "/lispdoc/cl-ppcre%3Ascan"
(lispdoc-path 'ql:quickload)
  ;; => "/lispdoc/quicklisp-client%3Aquickload"
```

Even if you find that perfectly acceptable, it is conceivable that we
had already published the original REST API to the world. So we need
to change the implementation without changing the interface. This is
where `uri-to-arguments` and `arguments-to-uri` might help:


```lisp
(defmethod uri-to-arguments ((resource (eql #'lispdoc)) uri)
  (multiple-value-bind (plain-args keyword-args)
      (call-next-method)
    (let* ((sym-name (string (first plain-args)))
           (package-name (or (cdr (assoc :package keyword-args)) 'cl))
           (sym (find-symbol sym-name package-name)))
      (unless sym
        (http-condition 404 "Sorry, no such symbol"))
      (values (cons sym (cdr plain-args))
              (remove :package keyword-args :key #'car)))))

(defmethod arguments-to-uri ((resource (eql #'lispdoc)) plain-args keyword-args)
  (let ((sym (first plain-args)))
    (call-next-method resource
                      (list sym)
                      (cons `(:package . ,(read-for-resource
                                           resource
                                           (package-name (symbol-package sym))))
                            keyword-args))))
```

We can now safely rewrite the remaining routes in much simpler fashion:

```lisp
(defun doc-not-found-message (symbol doctype)
  (let* ((othertype (if (eq doctype 'function) 'variable 'function))
         (otherdoc (documentation symbol othertype)))
    (with-output-to-string (s)
      (format s "There is no ~a doc for ~a." doctype symbol)
      (when otherdoc
        (format s "<p>But try <a href=~a>here</a></p>"
                (lispdoc-path symbol :doctype othertype))))))

(defroute lispdoc (:get :text/* (sym symbol) &key (doctype 'function))
  (or (documentation sym doctype)
      (http-condition 404 "No doc found for ~a" sym)))

(defroute lispdoc (:put :text/plain (sym symbol) &key (doctype 'function))
  (setf (documentation sym doctype)
        (payload-as-string)))

(defroute lispdoc (:get :text/html (sym symbol) &key (doctype 'function))
  (or (documentation sym doctype)
      (http-condition 404 (doc-not-found-message sym doctype))))

(defroute lispdoc (:put :application/json (sym symbol) &key (doctype 'function))
  (let* ((json (handler-case
                   (json:decode-json-from-string
                    (payload-as-string))
                 (error (e)
                   (http-condition 400 "Malformed JSON! (~a)" e))))
         (docstring (cdr (assoc :docstring json))))
    (setf (documentation sym doctype) docstring)))
```

Other backends
--------------

_Snooze_ is backend-agnostic. You can use [Clack][clack], which is a
good option, or anything else.

_Snooze_ offers with `make-clack-app` for quickly jumping into the
action, but doesn't "require" Clack in any sense. Here's a way to hook
_Snooze_ to Hunchentoot directly:

```
(defclass snooze-acceptor (hunchentoot:acceptor)
  ((snooze-bindings :initarg :snooze-bindings :initform nil :accessor snooze-bindings)))

(defmethod hunchentoot:acceptor-dispatch-request ((acceptor snooze-acceptor) request)
  (multiple-value-bind (code payload payload-ct)
      (let (;; Optional, but we do all the error catching ourselves
            ;; 
            (hunchentoot:*catch-errors-p* nil)
            (snooze:*backend* :hunchentoot))
        (progv
            (mapcar #'car (snooze-bindings acceptor))
            (mapcar #'cdr (snooze-bindings acceptor))
          (handle-request (hunchentoot:request-uri request)
                          :accept (hunchentoot:header-in :accept request)
                          :method (hunchentoot:request-method request)
                          :content-type (hunchentoot:header-in :content-type request))))
    (setf (hunchentoot:return-code*) code
          (hunchentoot:content-type*) payload-ct)
    (or payload "")))

(defmethod backend-payload ((backend (eql :hunchentoot))
                            (type snooze-types:text))
  (let ((probe (hunchentoot:raw-post-data)))
    (assert (stringp probe) nil "Asked for a string, but request carries a ~a" (type-of probe))
    probe))

(defvar *server* nil)

(defun stop ()
  (when *server* (hunchentoot:stop *server*) (setq *server* nil)))

(defun start (&rest args &key (port 5000))
  (stop)
  (setq *server* (hunchentoot:start
                  (apply #'make-instance 'snooze-acceptor
                         :port port
                         ;; in this example "homepage" name a resource
                         ;; to display when the URI is empty
                         :snooze-bindings `((*home-resource* . homepage))
                         args))))
```

Support
-------

To ask questions, report bugs, or just discuss matters open an
[issue][issues] or send me email.

[quicklisp]: http://quicklisp.org
[asdf]: http://common-lisp.net/project/asdf/
[hunchentoot]: https://github.com/edicl/hunchentoot
[sharp-lisp]: irc://irc.freenode.net/#lisp
[issues]: https://github.com/capitaomorte/snooze/issues
[caveman]: https://github.com/fukamachi/caveman#routing
[clack]: https://github.com/fukamachi/clack
[cl-rest-server]: https://github.com/mmontone/cl-rest-server
[restas]: http://restas.lisper.ru/en/manual/routes.html#routes
[ningle]: http://8arrow.org/ningle/)
[cl-who]: http://weitz.de/cl-who/#escape-string-all)
