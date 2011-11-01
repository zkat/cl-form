# Introduction

cl-form is a presentation-agnostic utility meant to simplify the process of validating web form
parameters, transforming the strings received into more useful Lisp values, and reporting errors, as
well as providing values that web forms can be repopulated with.

    (defun always-error (value)
      (check-field nil "No matter what VALUE is, this will error.")
      value)

    (defun as-integer (value &aux *read-eval*)
      (check-field (every #'digit-char-p value) "~A is not a valid integer." value)
      ;; Validators must return the final, intended field value.
      (parse-integer value))

    (defun field-length (value &key min-length max-length)
      (when min-length
        (check-field (>= (length value) min-length) "Field must be at least ~A characters long." min-length))
      (when max-length
        (check-field (<= (length value) max-length) "Field must be ~A characters long or shorter." max-length))
      value)

    (deform test-form ()
      ((:whatever #'field-length :min-length 1)
       (:error-field 'always-error)
       (:int-field 'as-integer)
       ((:my-list list) (lambda (val)
                          (mapcar #'as-integer val)))
       ((:my-array array) (lambda (val)
                            (remove nil val)))))

    (let ((form (make-form 'test-form '(("whatever" . "a")
                                        ("error-field" . "b")
                                        ("int-field" . "42")
                                        ("my-array[5]" . "5")
                                        ("my-array[0]" . "0")
                                        ("my-list" . "1")
                                        ("my-list" . "2")))))
      (assert (and (form-valid-p form)
                   (string= "a" (field-value form :whatever))
                   (null (field-value form :error-field))
                   (eql 42 (field-value form :int-field))
                   (equal '(1 2) (field-value form :my-list))
                   (equalp #("0" "5") (field-value form :my-array)))))

# Defining

Defining a form is a two-step process:

  1. Define validator functions. These functions should use `check-field` as assertions, and return a possibly-transformed value to be used as the field-value.
  2. Write a `deform` form that associates field names with these validators.

*[function]* `check-field test error-format &rest error-format-orgs`

  Used like an assertion. If TEST is NIL, validation will exit at this point (not evaluating
  subsequent forms in the validator function). ERROR-FORMAT and ERROR-FORMAT-ARGS will then be used
  to generate an error message to be returned by `field-error`
  
*[macro]* `deform name nil field-definitions`

  The syntax for FIELD-DEFINITIONS is: `(field-name validator-function &rest validator-args)`.
  
  VALIDATOR-FUNCTION can be any function that receives one argument (which will be the 'raw' value
  for the field -- in most cases a plain string). This function should return a validater, processed
  value to be returned by `field-value`.
  
  Additionally, VALIDATOR-ARGS can be provided to each individual field. These will be applied,
  after the raw field value, to the validator function. To clarify, (:field-name #'list 1 2 3) will
  result in (<raw-value> 1 2 3) being returned by the 'validator'.

  FIELD-NAME can be either a symbol (which names the field), or a literal list containing a symbol
  and one of the symbols `LIST` or `ARRAY`.
  
  If the second item in FIELD-NAME is the symbol `LIST`, then all items in the binding alist
  provided to `make-form` that share this field's name will be collected into a list, which will
  become the raw value of the field. This list is also what the validator for this field will
  receive, instead of a simple string.

  If the second item in FIELD-NAME is the symbol `ARRAY`, then the binding alist will be searched
  for keys matching `"<field-name>[<integer>]"`, and an array will be created containing those items
  at the provided indices. The array will be as long as the maximum `<integer>` value + 1. Any
  missing indices before this maximum item will be NIL. As with `LIST` fields, the array itself will
  be the raw-value of the field, and that array will be passed to the validator function.

*[special variable]* `*form*`

  When a validator function is called, this variable is bound to the current form object. At this
  point, the form object's raw values are already populated, and are accessible through
  `field-raw-value`. Both `field-error` and `field-value` will return NIL for all fields.

# Usage

In order to actually use defined 

*[function]* `make-form name &optional binding-alist`

  Creates an instance of a form named by NAME. If BINDING-ALIST is provided, the form object's
  fields are immediately populated and all validators are called.
  
  The keys in BINDING-ALIST are bound case-insensitively according to the symbolic field names
  defined in the form's `deform`.
  
  If no BINDING-ALIST is provided, the form is considered 'unbound. `field-value`,
  `field-raw-value`, and `field-error` can still be called, but will always return NIL. This can be
  useful for rendering functions that use form objects.
  
*[function]* `form-valid-p form`

  Returns NIL if any of the form's validators fail, or if the form is unbound.

*[function]* `form-errors form`

  For bound forms, returns an alist of `(field-name-symbol . "Error Message")` if any fields failed
  validation.
  
*[function]* `field-raw-value form field-name`

  For bound forms, returns the 'raw' value for a field. For non-list non-array fields, this will
  always be a string bound according to the field's name, or NIL. For list and array fields, this
  will be a list and an array of strings, respectively. Missing values for array fields will be
  populated with NIL.

*[function]* `field-value form field-name`

  For bound forms, returns the validated, possibly transformed value for this field. The specific
  value returned by this will be the value returned by the field's validator function.

*[function]* `field-error form field-name`

  Returns NIL if the field successfully validated, or a string containing an error message if
  validation for the field failed.
