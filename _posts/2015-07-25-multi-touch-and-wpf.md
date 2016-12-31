---
title:   "Multi-touch and WPF"
date:    2015-07-25 12:44:00 UTC
excerpt: "Research on circumventing a severe lag in WPF input events when multiple fingers are on the touch screen."
---

For the last three years I've been working with WPF, Microsoft's desktop UI framework.  I like it and will write about that seperately, but there are a few issues, the hardest of which I've faced is circumventing [a severe lag in input events](https://connect.microsoft.com/VisualStudio/feedback/details/782456/wpf-touch-event-fires-with-delay) when multiple fingers are on the touch screen.

### Disabling StylusLogic

WPF uses the Windows 7-onward RealTimeStylus API through PenIMC.dll to raise pen and touch events.

```
PenIMC.dll > StylusLogic > TouchDevice > Events
```

There are no external fixes for the lag in the StylusLogic classes, but we can stop them raising events by tricking them into thinking input devices have been disconnected using this [Microsoft-suggested workaround](https://msdn.microsoft.com/en-us/library/vstudio/dd901337(v=vs.90).aspx), updated to support .NET 4.5.2, during the Window.SourceInitialized event:

``` csharp
var inputManagerType = typeof(InputManager);

_stylusLogic = inputManagerType.InvokeMember("StylusLogic", BindingFlags.GetProperty | BindingFlags.Instance | BindingFlags.NonPublic, null, InputManager.Current, null);

_stylusLogicType = _stylusLogic.GetType();
_countField452 = _stylusLogicType.GetField("_lastSeenDeviceCount", BindingFlags.Instance | BindingFlags.NonPublic);

while (Tablet.TabletDevices.Count > 0)
{
    if (_countField452 != null)
        _countField452.SetValue(_stylusLogic, 1 + (int)_countField452.GetValue(_stylusLogic));

    int index = Tablet.TabletDevices.Count - 1;

    _stylusLogicType.InvokeMember("OnTabletRemoved", BindingFlags.InvokeMethod | BindingFlags.Instance | BindingFlags.NonPublic, null, _stylusLogic, new object[] { (uint)index });
}
```

Your window will still roughly respond to touches after executing the above code, but you'll find that no touch events are being raised - touch input is now only seen by your code as mouse events. WPF touch behaviours such as panning scroll viewers won't work, as they rely on touch events.

### Re-implementing TouchDevice

TouchDevice is an abstract class, so you implement your own instances then WPF will raise events and continue to manage properties like Focus for you.

```
? > TouchDevice > Events
```

You could base an implementation on any of:

- PenIMC.dll, which does not appear to be documented
- DirectInput, which I found poorly implemented by touch drivers
- RawInput, which [requires hardware-specific code](http://www.codeproject.com/Articles/381673/Using-the-RawInput-API-to-Process-MultiTouch-Digit)
- WM_POINTER* window messages, available from Windows 8
- WM_TOUCH window messages, available from Windows 7

Unless you're working with fixed hardware or brave enough to tackle PenIMC.dll you'll be processing window messages.

Note that the RealTimeStylus API prevents windows from receiving the WM_TOUCH and WM_POINTERUPDATE messages (although strangely not WM_POINTERDOWN and UP) but if you've run the code above then that's no longer a problem.

### Implementing using WM_TOUCH

WM_TOUCH messages won't be received by your window by default, but that's easy to enable:

``` csharp
TouchNativeMethods.RegisterTouchWindow(hWnd, TouchNativeMethods.TWF_WANTPALM);
```

Fixing all the WPF controls to observe them may seem a nightmare, but WPF provides the abstract TouchDevice class to do nearly all of the work for you.  Implementing TouchDevice instances that run from WM_TOUCH messages isn't really all that difficult:

``` csharp
var inputCount = wParam.ToInt32() & 0xffff;
var inputs = new TOUCHINPUT[inputCount];

if (GetTouchInputInfo(lParam, inputCount, inputs, TouchInputSize))
{
    for (int i = 0; i < inputCount; i++)
    {
       var input = inputs[i];
       var position = new Point(input.x * 0.01, input.y * 0.01));
       position.Offset(-window.Left, -window.Top);

       TouchDeviceEmulator device;
       if (!_devices.TryGetValue(input.dwID, out device))
       {
           device = new TouchDeviceEmulator(input.dwID);
           _devices.Add(input.dwID, device);
       }

       if (input.dwFlags.HasFlag(TOUCHEVENTF.TOUCHEVENTF_DOWN))
       {
           device.SetActiveSource(PresentationSource.FromVisual(window));
           device.Position = position;
           device.Activate();
           device.ReportDown();
       }
       else if (device.IsActive && input.dwFlags.HasFlag(TOUCHEVENTF.TOUCHEVENTF_UP))
       {
           device.Position = position;
           device.ReportUp();
           device.Deactivate();
           _devices.Remove(input.dwID);
       }
       else if (device.IsActive && input.dwFlags.HasFlag(TOUCHEVENTF.TOUCHEVENTF_MOVE))
       {
          device.Position = position;
          device.ReportMove();
       }
   }
}

CloseTouchInputHandle(lParam);
handled = true;
```

Most of the rest of the TouchDevice implementation is trivial, but here are a couple more useful  methods:

``` csharp
public override TouchPoint GetTouchPoint(IInputElement relativeTo)
{
    Point pt = Position;
    if (relativeTo != null)
        pt = ActiveSource.RootVisual.TransformToDescendant((Visual)relativeTo).Transform(Position);

    var rect = new Rect(pt, new Size(1.0, 1.0));
    return new TouchPoint(this, pt, rect, TouchAction.Move);
}

protected override void OnCapture(IInputElement element, CaptureMode captureMode)
{
    Mouse.PrimaryDevice.Capture(element, captureMode);
}
```

Once your WndProc calls this code for WM_TOUCH messages, WPF touch events are raised again and gesture support restored.

This won't quite restore multi-touch visual feedback on Windows 8, but the [SetWindowFeedbackSetting](https://msdn.microsoft.com/en-us/library/windows/desktop/hh802871(v=vs.85).aspx) function can help.