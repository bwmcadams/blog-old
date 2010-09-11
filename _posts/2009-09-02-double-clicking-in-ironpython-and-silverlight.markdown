---
title: Double Clicking in IronPython + Silverlight
tags: tech IronPython Silverlight Python
layout: post
---
Silverlight rather oddly lacks a double click event.  You can detect single click, but for double clicking you're on your own.  I found some examples for C#, but none for IronPython and had, a few months ago, ported some code I found [here](http://shemesh.wordpress.com/2009/03/05/silverlight-attach-a-double-click-to-any-object/) for IronPython.

I've been using it in application development and user testing for about 6 months and it works fairly well in both Silverlight 2 and 3, although YMMV.
 
{% highlight python %} 
CLICK_INTERVAL = 500
 
class ClickHandler(object):
    """Attach a Double Click handler to any given UIElement,
    as by default Silverlight lacks support for double clicking.
    """
    _lastClickTime = 0
    _callback = None
 
    @staticmethod
    def attachDoubleClickHandler(target, callback):
        return ClickHandler(target, callback)
 
    def __init__(self, target, callback):
        target.MouseLeftButtonUp += self.target_MouseLeftButtonUp
        target.MouseLeftButtonDown += self.target_MouseLeftButtonDown
        self._callback = callback
 
    def target_MouseLeftButtonUp(self, sender, args):
        self._lastClickTime = DateTime.Now.Ticks / 10000
 
    def target_MouseLeftButtonDown(self, sender, args):
        now = DateTime.Now.Ticks / 10000
        if now - self._lastClickTime < CLICK_INTERVAL and \
          now - self._lastClickTime > 25:
            self._callback(sender, args)
{% endhighlight %}
 
You may want to tweak the click interval based on what works for you.  500 milliseconds worked well with my users, for the application in question. To attach a ClickHandler simply invoke the static method `ClickHandler.attachDoubleClickHandler(«uiElementObj», «callbackFunc»)`

{% highlight python %}
from System import Uri, UriKind
from System.Windows import Application, UIElement
from System.Windows.Browser import HtmlPage
 
from util import ClickHandler
 
 
class Foo(UIElement):
  """Sample UIElement class demonstrating usage of 
  ClickHandler and it's events.
  """
 
  def __init__(self):
    Application.LoadComponent(self, Uri('xaml/widgets/foo.xaml',
            UriKind.Relative))
    ClickHandler.attachDoubleClickHandler(self, self.evt_dblClicked)
 
  def evt_dblClicked(self, sender, args):
    """Event for double click; this gets passed
    the original "MouseLeftButtonDown" arguments 
    which ClickHandler got, so you'll want to match
    the "MouseButtonEventHandler" delegate arguments.
    Object sender - the object the mouse click handler is attached to,
            in this case, should be the same as 'self'
    System.Windows.Input.MouseButtonEventArgs args - the mouse event arguments
    """
    HtmlPage.Window.Alert('%s got double clicked!' % sender)
{% endhighlight %}
 
It's not the world's greatest solution, but it gets the job done.

