+++
date = "2017-07-07T14:58:33+02:00"
description = "GSoC blog series on rewriting Piper"
tags = [ "Piper", "fdo", "MouseMap", "resolutions", "pygi", "template" ]
title = "GSoC part 7: the resolutions stack page"
categories = "Development"
series = "Google Summer of Code"
+++

![GSoC logo horizontal](/img/blog/gsoc-part-1/GSoC-logo-horizontal.svg)

This week was a good week with loads of progress. I'll just go through each item
one by one, as usual.

### MouseMap merged

Last Friday I opened the pull request to merge the MouseMap widget. After a bit
of a discussion around preferences, my mentor and I decided to merge ongoing
work in a `rewrite` branch. This keeps the master branch functional (while still
receiving improvements, such as adding new functionality to the ratbagd bindings
or [a Gtk.Application subclass][gtkapplication]), but it also prevents reviewing
the same pieces of code over and over until they finally end up in master.

With the MouseMap finally done, it was time to get started on the real work:
implementing the mockups!

### pygi_composite_templates, or: using Gtk+ composite templates for Python

GtkBuilder [templates][templates] allow you to define your composite (composed
of other widgets) widget through `.ui` files, which are XML files generated by
[Glade][glade].  This is nice in that it removes all the boilerplate code that
creates widgets, sets their properties and connects their signals. Instead, you:

1. Call `gtk_widget_init_template` in your widget's `_init` function;
2. Call `gtk_widget_set_template` or `gtk_widget_set_template_from_resource` in your widget's `_class_init` function;
3. Call `gtk_widget_class_bind_template_child` (or one of its variants) as needed for all members that you need to access.

Nice right? Well, [not for Python][bugzilla]. That is, not officially:
[`pygi_composite_templates`][gi_composite] is prototype implementation of Gtk+
composite widget templates for PyGI. In an experiment I've tried how this works
for Piper, and fortunately I haven't noticed any issues yet! Thanks to templates
the UI code is now significantly cleaner and easier to comprehend, not to
mention there is much less of it as well.

There was only one issue with `gi_composite` when I tried it: it wouldn't find
Piper's `.ui` file, even though it was declared in `piper.gresource.xml` which
gets added to [Gio's resources][gio-resource]. I contacted Dustin,
`gi_composite`'s author, who kindly responded:

> One thing you might try is ensuring that your resource is registered *before*
> your ui class type is created -- note that I said type, not instance. If I
> recall correctly, I think the ui file may be needed when the class type is
> created... which is when the interpreter executes the line with 'class Foo' on
> it.

And indeed, [shuffling][shuffling] some lines around so that the resource gets added before
anything else does the trick! The complete implementation of this can be found
in [this][gi_composite-commit] commit.

### The resolutions page

This all paved the way to implementing the first page of the mockups: the
resolutions page! In doing so I encountered multiple little obstacles that I
resolved along the way, but let's first run through the resolutions page.

The UI elements are all defined in `.ui` files, keeping the code blissfully
clean (trust me \*Star Wars wave\*). In fact, the complete Python code weighs in
at just around 200 lines. So, what does it do? Well, this:

<video controls>
  <source src="/img/blog/gsoc-part-7/resolutions.webm" type="video/webm">
Your browser does not support the video tag.
</video>

In case committing the changes to the device doesn't succeed, there's also an
[in-app notification][in-app] that informs the user as such:

![In-app notification](/img/blog/gsoc-part-7/notification.png)

#### Obstacle 1: no support for adding or removing resolutions

According to the [mockups][mockups], Piper should be able to add and remove (or
enable and disable, if the device doesn't support this) resolutions. It turns
out, however, that [this isn't yet supported by libratbag][profiles]. The
buttons are there, and until this is supported they will just print a message to
`stdout`.

#### Obstacle 2: a commit method

While working on the above changes, I noticed that any changes made were not
saved on my device. Then I remembered that libratbag has a commit-style API: any
changes made are not stored on the device unless explicitly committed. Ratbagd,
however, didn't expose this method yet, so I had to add it.

The ratbagd PR is [here][commit-ratbagd], and the Piper bindings commit is
[here][commit-piper].

#### Obstacle 3: error codes, and using the correct resolution setter

Once the commit method had been added everywhere, my changes *still* weren't
saved on the device (sidenote: I did find [this bug][resolution-signal]).
Benjamin helped me in figuring out why; it turned out that ratbagd
unconditionally used the `ratbag_resolution_set_dpi_xy` method, which is meant
for devices that support separate x- and y-resolutions, which my device doesn't.
Benjamin [fixed][ratbagd-cap-xy] this, and I added support for libratbag's
[error codes][commit-piper] to Piper's ratbagd bindings such that these cases
are noticed in the future (yes, this and the commit method were solved
simultaneously).

#### Obstacle 4: does every resolution have its own report rate?

With the resolution list working, I started on the report rate. The
[mockups][mockups] show that the report rates work per profile, across all that
profile's resolutions. Ratbagd's DBus API, however, has a `SetReportRate` method
for every resolution. To discuss this I opened [this issue][report-rate]; the
concensus for now is to support the per profile report rate. This isn't
implemented in ratbagd ([yet?][report-rate-profile]), so for now Piper is the
odd one out by making sure that a changed report rate is propagated to all a
profile's resolutions.

#### Obstacle 5: drawing only leaders that have children

Before this week, the MouseMap unconditionally drew all the leaders in the SVG.
The resolutions page needs to indicate which button(s) is (are) assigned to
switch resolutions, so it adds less children than there are buttons on the
device (unless every button is assigned to switch resolutions...). To draw only
leaders that have children, the paths and their origin square in the SVG are
grouped and this group is given the identifier `buttonX-path` (or `ledX-path`),
and the MouseMap's `_draw_device` method is adjusted:

```diff
@@ -391,4 +398,6 @@ def _draw_device(self, cr):
              self._handle.render_cairo_sub(highlight_context,
                                            self._highlight_element)
              cr.mask_surface(highlight_surface, 0, 0)
 -        self._handle.render_cairo_sub(cr, id=self._layer)
 +        for child in self._children:
 +            self._handle.render_cairo_sub(cr, id=child.svg_path)
 +            self._handle.render_cairo_sub(cr, id=child.svg_leader)
```

Finally, to find buttons that are assigned to switch resolutions:

```python
mousemap = MouseMap("#Buttons", self._device, spacing=20, border_width=20)
for button in profile.buttons:
    if button.action_type == "special" and button.special == "resolution-default":
        label = Gtk.Label(_("Switch resolution"))
        mousemap.add(label, "#button{}".format(button.index))
```

#### Obstacle 6: keep scrolling the list when the cursor hovers a scale

As you can see in the video, the list contains scales (<q>sliders</q>). If, when
scrolling the list, the cursor hovers a scale, the scale would consume the
scroll event. Not nice. To work around this, we capture the scroll event and
block the signal on the scale:

```python
@GtkTemplate.Callback
def _on_scroll_event(self, widget, event):
    # Prevent a scroll in the list to get caught by the scale
    GObject.signal_stop_emission_by_name(widget, "scroll-event")
    return False
```

#### Obstacle 7: adjusting the scale in multiples of 50 DPI

Most mice only support a resolution in multiples of 50 DPI; anything less than
that doesn't make much sense. The `Gtk.Scale` wouldn't adhere to its
`Gtk.Adjustment`'s `step-increment` property, even though it was set. Weirdly,
it does adhere to the `page-increment` property. After asking around on `#gtk+`,
I ended up connecting to the `change-value` signal, which is emitted when a
scroll action is performed on a range (a [scroll action][scroll-type] can be
anything, really). The signal handler receives the new value, which we can round
to the nearest multiple of 50 before setting it on the scale:

```python
@GtkTemplate.Callback
def _on_change_value(self, scale, scroll, value):
    # Round the value resulting from a scroll event to the nearest multiple
    # of 50. This is to work around the Gtk.Scale not snapping to its
    # Gtk.Adjustment's step_increment.
    scale.set_value(int(value - (value % 50)))
    return True
```

### Theming in ratbagd

This week ratbagd [got][ratbagd-themes] support for different SVG themes. The
reasoning behind this is that Piper, an application intended to integrate with
the GNOME Desktop Environment, prefers a different style of SVGs than a
hypothetical KDE version (Kiper? &#128521;) would.

The linked PR adds a new DBus property
`org.freedesktop.ratbagd1.Manager.Themes`, which returns a string array of
themes installed with libratbag; currently `default` and `gnome`.
`org.freedesktop.ratbagd1.Device` got a new method `GetSvg(string)`, which takes
a theme and returns the full path to the device SVG for the given theme.

There was a [long lasting PR][toned-down-svg] open on libratbag to convert all
SVGs to the toned down style as preferred by Piper (together with issues
[178][178] and [171][171]). This PR was rebased on top of the theme changes,
after [moving the default SVGs to their theme directory][default-svg], which
probably wasn't done in the theme PR as an oversight.

The GNOME theme currently sports three device SVGs. If you are familiar with
Inkscape and willing to do a few; feel free! [Here][gnome-svg] is the list of
devices that aren't yet found in the GNOME theme.

Today I quickly [added][ratbagd-themes] theme support to Piper's ratbagd
bindings, so that the MouseMap can be adjusted and everything will work with the
new libratbag master.

The changes required to the MouseMap are rather straightforward:

```diff
--- a/piper/mousemap.py
+++ b/piper/mousemap.py
@@ -106,7 +106,8 @@ class MouseMap(Gtk.Container):
             raise ValueError("Layer cannot be None")
         if ratbagd_device is None:
             raise ValueError("Device cannot be None")
-        if not os.path.isfile(ratbagd_device.svg_path):
+        svg_path = ratbagd_device.get_svg("gnome")
+        if not os.path.isfile(svg_path):
             raise ValueError("Device has no image or its path is invalid")

         Gtk.Container.__init__(self, *args, **kwargs)
@@ -118,8 +119,8 @@ class MouseMap(Gtk.Container):
         self._children = []
         self._highlight_element = None

-        self._handle = Rsvg.Handle.new_from_file(ratbagd_device.svg_path)
-        self._svg_data = etree.parse(ratbagd_device.svg_path)
+        self._handle = Rsvg.Handle.new_from_file(svg_path)
+        self._svg_data = etree.parse(svg_path)

         # TODO: remove this when we're out of the transition to toned down SVGs
         device = self._handle.has_sub("#Device")
```

That was it for this week! You can find the complete resolutions page PR
[here][resolution-pr].  Hopefully it will be merged early next week, so that I
can start on the next page.

This blog post is part of a [series](/series/google-summer-of-code/). You can read the next part about the LED
stack page [here](/blog/gsoc-part-8) or the previous part
[here](/blog/gsoc-part-6).

[gtkapplication]: https://github.com/libratbag/piper/pull/24
[templates]: https://wiki.gnome.org/HowDoI/CustomWidgets#Templates
[glade]: https://glade.gnome.org/
[bugzilla]: https://bugzilla.gnome.org/show_bug.cgi?id=701843
[gi_composite]: https://github.com/virtuald/pygi-composite-templates/
[gio-resource]: https://github.com/libratbag/piper/blob/master/piper.in#L48
[shuffling]: https://github.com/libratbag/piper/pull/28/commits/cbc546dc945f9bbdbf4bd3671dc25219d3a3208b#diff-07d882117a676ac39c6d2cee78a8876aR40
[gi_composite-commit]: https://github.com/libratbag/piper/pull/28/commits/cbc546dc945f9bbdbf4bd3671dc25219d3a3208b
[in-app]: https://developer.gnome.org/hig/stable/in-app-notifications.html.en
[mockups]: https://github.com/libratbag/piper/raw/wiki/redesign/resolution.png
[profiles]: https://github.com/libratbag/libratbag/issues/211
[commit-ratbagd]: https://github.com/libratbag/libratbag/pull/199
[commit-piper]: https://github.com/libratbag/piper/pull/26
[resolution-signal]: https://github.com/libratbag/libratbag/issues/205
[ratbagd-cap-xy]: https://github.com/libratbag/libratbag/pull/201
[report-rate]: https://github.com/libratbag/libratbag/issues/197
[report-rate-profile]: https://github.com/libratbag/libratbag/issues/202
[scroll-type]: https://lazka.github.io/pgi-docs/#Gtk-3.0/enums.html#Gtk.ScrollType
[ratbagd-themes]: https://github.com/libratbag/libratbag/pull/209
[toned-down-svg]: https://github.com/libratbag/libratbag/pull/182
[178]: https://github.com/libratbag/libratbag/issues/178
[171]: https://github.com/libratbag/libratbag/issues/171
[default-svg]: https://github.com/libratbag/libratbag/pull/212
[gnome-svg]: https://github.com/libratbag/libratbag/issues/213
[ratbagd-themes]: https://github.com/libratbag/piper/pull/29
[resolution-pr]: https://github.com/libratbag/piper/pull/28
