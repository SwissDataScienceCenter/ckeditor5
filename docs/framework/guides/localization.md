---
category: framework-guides
order: 50
---

# Localization Guide

## Introduction

The CKEditor 5 ecosystem support message localization. It means that the UI of any feature can be translated for various languages and regions depending on the user's preferences. With the  CKEditor 5 at version 19.0.0 and CKEditorWebpackPlugin at version 9.0.1, we opened the translation system for 3rd-party plugins, provided a way for adding missing or fixing invalid translations to the editor and we introduced support for translating plural forms.

### Open API

Until now the building process was creating short strings for message IDs. This behavior was confusing, it caused non-deterministic builds (as the ID was generated during the webpack compilation) and made adding translations to the build impossible as IDs have been changing with each new CKEditor 5 release and with each project rebuilding. So we decided to drop the message ID shortening feature, as the gain from lower bundle size was disproportionately lower than the loss from the mentioned issues. That means, that from now:

- All missing translations can be added directly to the build without worrying about future releases and rebuilding the editor. Though, we still highly recommend adding missing or incorrect strings via [https://www.transifex.com/ckeditor/ckeditor5/](https://www.transifex.com/ckeditor/ckeditor5/) as we fetch translations from there with every release.
- building the editor two times with [CKEditorWebpackPlugin](https://github.com/ckeditor/ckeditor5-dev/tree/master/packages/ckeditor5-dev-webpack-plugin) will produce the same output
- plugin owners will be able to use the `t()` ([https://ckeditor.com/docs/ckeditor5/19.0.0/api/module_utils_locale-Locale.html#function-t](https://ckeditor.com/docs/ckeditor5/19.0.0/api/module_utils_locale-Locale.html#function-t)) function and easily provide translations for messages in a few ways

### Glossary

Before we start, let's explain what the crucial part od the translation process mean:

- *message* - a string or an object that should be translated, the string version works as a shortcut for the `{ id: message, string: message }` object form,
- *message ID* - a property used to distinguish messages, useful for short messages, where a collision can occur, like `%0 images`,
- *message string* - the default (English) form of the message, the default singular version when the message is supposed to support plural versions,
- *message plural* - an optional plural (English) version of the message, the presence of this property indicates that the message should support both, singular and plural forms,
- PO file (.po) A file containing translations for the language in the [PO format](https://www.gnu.org/software/gettext/manual/html_node/PO-Files.html). All
- POT file (.pot) A file containing source messages (English sentences) that will be translated,
- `contexts.json` files - Files including *message contexts* used by the CKEditor 5 team to provide additional information for translators for messages used in the code.
- CKEditor 5 translation asset - a JS file added to the generated bundle or generated as a separate file during the webpack build step.

## Writing localizable UI

All *messages* needing localization should be passed to the special CKEditor 5's `t()` function. This function can be retrieved from the editor's `locale` instance: `const { t } = editor.locale;` or from any view method `const t = this.t;` Note that this function should not be used as the `Locale` class method because during the translation process a static code analyzer catches translation messages only from function forms.

The `t()` function as the first argument accepts either a string literal, which will be at the same time the message ID and the message string or an object literal containing `id`, `string`, and optional `plural` property. This function can't be called on variables or other expressions for the same reason as above.

As the second argument, the translation function accepts a value or an array of values. These "values" will be used to fill the placeholders in the more advanced translation scenarios. And if the `plural` property is specified then the first value will be used as quantity determining the plural form.

When using the `t()` function, you can create your own messages or reuse the messages created in CKEditor 5 packages your project depends on. In case of reusing messages, you won't have to worry about translating them as all work will be done by the CKEditor 5 team and Transifex translators anyway (but your help in translating still will be appreciated).

For simple messages use the string form for simplicity:

```js
t( 'insert emoji' );
t( 'insert %0 emoji', emojiName );
t( 'insert %0 emoji', [ emojiName ] );
```

For more advanced scenarios use plain objects forms:

```js
t( { string: '%0 emoji', id: 'INSERT_EMOJI' } );
t( { string: '%0 emoji', plural: '%0 emojis' }, quantity );
t( { string: '%0 emoji', plural: '%0 emojis', 'INSERT_EMOJI' } );
```

### Examples

### Localizing UI of plugins:

```js
// ...
editor.ui.componentFactory.add( 'smilingFaceEmoji', locale => {
	const command = editor.commands.get( 'insertSmilingFaceEmoji' );
	const buttonView = new ButtonView( locale );

	// The translation function.
	const { t } = editor.locale;

  // The localized label.
	const label = t( 'insert smiling face emoji' );

	buttonView.set( {
		label,
		icon: emojiIcon,
		tooltip: true
	} );

	buttonView.on( 'execute', () => {
		editor.execute( 'insertSmilingFaceEmoji' );
		editor.editing.view.focus();
	} );
} );
// ...
```

Note that this sample lacks a few parts. To check how to create a complete plugin, for example, check the [Creating a simple plugin guide](https://ckeditor.com/docs/ckeditor5/latest/framework/guides/creating-simple-plugin.html).

### Localizing aria attributes:

```js
editingView.change( writer => {
	const editingView = this._editingView;
	const t = this.t;

	editingView.change( writer => {
		const viewRoot = editingView.document.getRoot( this.name );

		writer.setAttribute( 'aria-label', t( 'Rich Text Editor, %0', [ this.name ] ), viewRoot );
	} );
} );
```

### Localizing pending actions

Pending actions are used to inform the user that the action is in progress and we will lost data while exiting the editor - see [https://ckeditor.com/docs/ckeditor5/latest/api/module_core_pendingactions-PendingActions.html](https://ckeditor.com/docs/ckeditor5/latest/api/module_core_pendingactions-PendingActions.html)

```js
_updatePendingAction() {
	// ...
	const pendingActions = this.editor.plugins.get( PendingActions );

	// ...
	const t = this.editor.t;
	const getMessage = value => `${ t( 'Upload in progress' ) } ${ parseInt( value ) }%.`;

	this._pendingAction = pendingActions.add( getMessage( this.uploadedPercent ) );
	this._pendingAction.bind( 'message' ).to( this, 'uploadedPercent', getMessage );
}
```

## Adding translations and localizing the editor UI

First of all, if you have found a missing or an incorrect translation in the CKEditor 5 translations, check [how you can contribute to the project](https://ckeditor.com/docs/ckeditor5/18.0.0/framework/guides/contributing/contributing.html#translating)! Your help will be appreciated by others!

Adding translations to the editor can be done in three ways to satisfy various needs.

- by adding translations via the `[translations-service#add()](https://ckeditor.com/docs/ckeditor5/19.0.0/api/module_utils_translation-service.html#static-function-add)` function - this function can be used at runtime before the editor starts
- by extending the global `window.CKEDITOR_TRANSLATIONS` object - this object can be modified before and after the build step. CKEditor 5 translations assets operate on this object.
- by creating `.po` files with translations in the `lang/translations/` directory of the published package like other CKEditor 5 packages do - this option will be useful for 3rd-party plugin creators as this allows bundling translations only for needed languages during the webpack compilation.

Note that the [CKEditorWebpackPlugin](https://github.com/ckeditor/ckeditor5-dev/tree/master/packages/ckeditor5-dev-webpack-plugin) is configured to parse by default only the CKEditor 5 source code when looking for messages and attaching translations to build. If you develop your own plugin outside of CKEditor 5 ecosystem you should override the `sourceFilesPattern` option (and the `packageNamePattern` option if you are creating `.po` files) to allow `CKEditorWebpackPlugin` to analyze the code and find messages to translate (and `.po` files with available translations). You should also mention these webpack plugin changes in your readme to make other users build the localized CKEditor 5 editor with your plugin correctly. This obstacle may be simplified in the future when the localization feature gets more popular.

The first option of adding translations is via the  [`translations-service#add()`](https://ckeditor.com/docs/ckeditor5/19.0.0/api/module_utils_translation-service.html#static-function-add) helper. This utility adds translations to the `window.CKEDITOR_TRANSLATIONS` object by extending it. Since it needs to be imported It works only before building the editor. With the recent release, it started accepting the `getPluralForm` function as the third argument and started accepting an array of translations for a message if the message should support singular and plural forms.

```js
add( 'pl', {
  'Add space': [ 'Dodaj spację', 'Dodaj %0 spacje', 'Dodaj %0 spacji' ]
} );
```

In the case of adding translations via the `window.CKEDITOR_TRANSLATIONS` object, for each language that should be supported, the `dictionary` property should be extended and the `getPlrualForm` function should be provided if missing. The `dictionary` property is a `message ID ⇒ translations` map, where the `translations` can be one sentence (a string) or an array of translations with plural forms for given language if the string should support singular and plural forms. The `getPluralForm` property should be a function that for given quantity returns the plural form index. Note that while using CKEditor 5 translations this property will be defined by CKEdtior5 translation assets. Let's check an example for adding Polish translations below:

```js
{
	 // Each key should be a valid language code.
   pl: {
			// A map of translations for the 'pl' language.
      dictionary: {
         'Cancel': 'Anuluj',
         'Add space': [ 'Dodaj spację', 'Dodaj %0 spacje', 'Dodaj %0 spacji' ]
      },
      // A function that returns the plural form index for the given language.
      // getPluralForm: n => n == 1 ? 0 : n % 10 >= 2 && n % 10 <= 4 && ( n % 100 < 10 || n % 100 >= 20 ) ? 1 : 2
   }
   // Other languages...
}
```

If you add a new language, remember to set the `getPluralForm` function which should return a number (or a boolean for languages with simple plural rules like English) that indicates which form should be used for which value.

The 3rd option of adding plugins should fit mostly plugin owners containing many localizable messages. Using that option you need to create a `.po` file per each language code in the `lang/translations/` directory containing translations that follow the [PO file format](https://www.gnu.org/software/gettext/manual/html_node/PO-Files.html).

```
# lang/translations/es.po

msgid ""
msgstr ""
"Language: es\n"
"Plural-Forms: nplurals=2; plural=(n != 1);\n"

msgid "Align left"
msgstr "Alinear a la izquierda"
```

To build the localized editor follow the steps from [building the editor using a specific language guide](https://ckeditor.com/docs/ckeditor5/18.0.0/features/ui-language.html#building-the-editor-using-a-specific-language).

### Known limitations

- It's impossible to change the chosen language at runtime without destroying the editor.
- It's impossible to add more than one language to the bundle using the `CKEditorWebpackPlugin`.
