# About Text Handling in iOS

The iOS platform gives you many ways to display text in your apps and let users edit that text. It also lets you display formatted text and web content in your app’s views. The resources at your disposal range from framework objects—such as text views, text fields, and web views—to text layout engines that you can use directly to draw, lay out, and otherwise manage text.

![](image/0/textpg_intro_2x.png)

With the classes in the UIKit framework, you can manage the edit menu (including adding custom items to it), implement custom input views, and copy, cut, and paste data within and between apps.

> Note: This document was previously titled Text, Web, and Editing Programming Guide for iOS.

### At a Glance

Apps in iOS have a number of powerful technologies to handle text, both for editing text and for rendering high-quality typographically formatted text.

##### The UIKit Framework Provides Your App with Text and Web Objects

You can add ready-made text views, text fields, and labels to your app’s user interface by using instances of the UITextView, UITextField, and UILabel. You can add and configure them programmatically or by using the Interface Builder editor in Xcode. You can also turn a view of your app into a miniature web browser capable of understanding and displaying HTML, CSS, and JavaScript content.

> Relevant Chapters: Displaying Text Content in iOS, Typographical Concepts, Managing Text Fields and Text Views

##### When Users Edit Text, Your App Must Manage the Keyboard

When a user taps a text field, text view, or form field in a web view, iOS animates a keyboard into view. An app can control which keyboard is presented; for example, for a numeric-value field, the app should select the numeric keypad. If the entered or edited text is obscured by the keyboard, the app should adjust the view displaying the text so that the text appears above the keyboard. The delegate of a text view, text field, or web view is responsible for validating edited text and for accessing and storing edited text when the user dismisses the keyboard.

> Relevant Chapters: Managing the Keyboard

Underlying the text views in UIKit is a powerful layout engine called Text Kit. If you need to customize the layout process or you need to intervene in that behavior, you can use Text Kit. Text Kit is a set of classes and protocols that provide high-quality typographical services which enable apps to store, lay out, and display text with all the characteristics of fine typesetting, such as kerning, ligatures, line breaking, and justification.

For most apps, you can use the high-level text display classes and Text Kit for all their text handling. For smaller amounts of text and special needs requiring custom solutions, you can use alternate, lower level technologies, such as the programmatic interfaces from the Core Text, Core Graphics, and Core Animation frameworks as well as other APIs in UIKit itself.

To communicate directly with the text-input system of iOS, implement the UITextInput protocol and related protocols and classes. Your app can also make use of technologies for spell checking and regular expressions.

> Relevant Chapter: Using Text Kit to Draw and Manage Text, Lower Level Text-Handling Technologies

