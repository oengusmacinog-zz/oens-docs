******************
Conversion Process
******************

As we are converting components over to mergeStyles we have come to the realization that it's just too much work to do the entire conversion in one PR. After working with a few folks on our team it's clear that when we are working on the conversion we should follow the below process.

Pull Requests
=============

To make life easier for folks reviewing our pull requests we need to break up the conversion into three separate parts.

Part One
--------

We want to build out the mergeStyles scaffolding for each component, move the main component logic and styles to ComponentName.base.tsx and open a pull request for review before even worrying about converting the SCSS to mergeStyles.

Part Two
--------

This part is a bit optional but if the component has either deeply nested styles or misuse of global styles, it should be cleaned up in this step and a PR should be opened.

Part Three
----------

Convert all styles to mergeStyles and open a PR. At this point, the component accepts mergeStyles and SCSS and now we need to convert over styles and remove the old SCSS file.

Common Roadblocks
=================

Applying a Style Set to a Sub-component That Already Has a Root Style Set
-------------------------------------------------------------------------

When applying style sets in a component (e.g. Nav) to a sub-component (e.g. ActionButton) which already has a root style set you will run into an issue using a `$` selector on the item like:

.. code-block:: JavaScript

    '$link:hover &': {
        color: blue
    }

The conflict happens because it's root style set would normally be applied to a class like **root-###**, but being applied as `link` inside the Nav component applies that style set to **link-###** class name. Styles applied using this `$` selector syntax do not render out. This is an open bug [here](https://github.com/OfficeDev/office-ui-fabric-react/issues/4138).

Until then the only option is to use semantic classNames to target these elements.

.. code-block:: JavaScript

    '.ms-Link:hover &': {
        color: blue
    }


### Static Functions Do Not Work in a Decorated Class
-----------------------------------------------------

Part of the process of converting a component to MergeStyles is to add the @customizable decorator to the ComponentBase class to allow theming to work.  This will, unfortunately, break any usage of static functions.

For example, the Layer component has a sibling component, LayerHost, that uses a static function from LayerBase.

.. code-block:: JavaScript

    LayerBase.notifyHostChanged(this.props.id);


When LayerBase is using the @customizable decorator then the Class is no longer ``LayerBase`` but the outputted Class of :class:`ComponentWithInjectedProps` which itself does not possess the static class **notifyHostChanged**, so you will get a *Not a Function* error.  There is a bug open [here](https://github.com/OfficeDev/office-ui-fabric-react/issues/3988).

In the meantime there are two possible options:

1. **Don't apply @customizable decorator** - This is not a good choice as we want to achieve global theme application.  However, in the example case of Layer, there are not actually any themeable styles - colors, fonts, etc - the component is just structural with a little bit of CSS to determine the layout. In this case, the @customizable decorator was skipped.

1. **Convert static class functions to regular exported functions** - The difference between static class functions and regular exported functions is minimal with the big difference being how you handle the Class data so this should be possible to do, but could still be problematic.  Hopefully, this issue on Github will eliminate this soon.

Tests Fail
----------

The change in file structure and the @customizable decorator will make certain types of tests fail that might need tweaking to make sure the tests are still looking for the right Class.  Especially tests that don't fully mount the component to test a very specific case will likely fail.

For example using enzyme's `shallow()` function will no longer find the right class with **ComponentBase** nested inside or **Component** nested inside Customizable's **ComponentWithInjectedProps**. The common folder now contains a new helper function `shallowUntilTarget()` you can use instead. You can find the documentation on how to use `shallowUntilTarget()` [here](Testing#test-utilities--helpers).

Guidelines
==========

Component.types.ts
------------------

IComponentStyles interface
^^^^^^^^^^^^^^^^^^^^^^^^^^

* The properties in this interface should all be required. E.g. `root: IStyle;` and not `root?: IStyle;`

ClassNames logic
----------------

Any logic for determining a component or element's classNames should reside in the Component.styles.ts file. This may mean getting rid of a few utility/state classNames in favor of props to add styles in conditionally.

Example - This sass block has a few syntax specifics and a state className to change styles, and the component combined it with the className `ms-Check`. We need the component to only call one className `className={ classNames.root }`:

::

    .root {
        line-height: 1;
        height: $checkBoxHeight;

        &:before {
            content: '';
            background: $bodyBackgroundColor;
        }

        &.rootIsChecked:before {
            background: $ms-color-themePrimary;
            @include high-contrast {
            background: Window;
            }
        }
    }


Here is what the resulting conversion should look like:

.. code-block:: TypeScript

    root: [
        'ms-Check', // Add in the className you want as a string

        { // Open your styles block
            lineHeight: '1',
            height: checkBoxHeight, // checkBoxHeight comes from IComponentStyleProps but is set to a default value in the prop deconstructor.

            selectors: {
            '&:before': {
                content: '""',
                background: semanticColors.bodyBackground, // semanticColors comes from theme prop.
            }
            }
        },

        checked && [ // checked comes from IComponentStyleProps as a boolean.
            'is-checked',
            {
            selectors: {
                '&:before': {
                background: palette.themePrimary, // palette comes from theme prop.
                selectors: {
                    [HighContrastSelector]: { // Styling library contains many useful replacements for old mixins.
                    background: 'Window'
                    }
                }
                }
            }
            },
        ],
        className // className comes from props and allows developers to add their own className to the root element.
    ],

Common Fabric Snippets
----------------------

There is a help VSCode extension containing snippets of commonly used Fabric and mergeStyles code here:
https://marketplace.visualstudio.com/items?itemName=jordanjanzen.office-ui-fabric-react-snippets.

You can also install it from the Extensions panel in VSCode.

The extension readme has instructions on how to use it, but the snippets do assume a few things so it's best to review after use and make sure to remove any unnecessary code or move it into the appropriate places.

Exports
-------

As we want the end result to be entirely themeable, it's best to export all your created component.base components so developers can create their own styles.

Styling best practices
======================

Put selectors last
------------------

While the order of properties generally doesn't matter (alphabetical is a fair default if you have no other preference), the `selectors` property should come last. This improves readability by preventing a single property from 'hiding' below a large `selectors` property.

.. code-block:: TypeScript

    element: {
        color: 'blue',
        margin: 0,
        overflow: 'inherit',
        padding: 0,
        textOverflow: 'inherit',
        selectors: {
            '&:hover': {
            color: 'red',
            margin: 10
            }
        }
    }
