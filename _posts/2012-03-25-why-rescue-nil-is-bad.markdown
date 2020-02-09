From time to time I see people write code and at the end of the line there will be: _**rescue nil**_. This is not good because it will rescue from **any** exception.

I see people rescuing nil because they have a variable and they aren't sure if it is nil so they want to protect against running a method on it which would cause an error.
The two most common rescue nil scenarios I see people protect against are:

*   a parameter not being present
*   some method being called on a variable that may represent an object or may be nil

Below is an example of rescue nil that I see often. The rescue nil on line 13 is there to protect against a category not being present. This rescue will also handle an exception due to the caching system not being available. If you have any notification systems in place such as [Airbrake](http://www.airbrakeapp.com) or [exception_notification](https://github.com/smartinez87/exception_notification) you aren't going to hear about your caching system system being down. If there is some other error in the _ask_question_ method you also are not going to hear about it because rescue nil will eat everything up.
<script src="https://gist.github.com/2197335.js?file=rescue_nil_bad.rb"></script>

A better solution is to just check and see if the category is actually present before running a method on it.

<script src="https://gist.github.com/2197440.js?file=rescue_nil_bad_fixed.rb"></script>
Thanks for reading, Let me know your thoughts.

## Related Links

*   [Airbrake](http://airbrakeapp.com)
*   [Exception Notifier gem](https://github.com/smartinez87/exception_notification)
