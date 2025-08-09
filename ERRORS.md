Based on the screenshot, here are the error messages displayed in the text editor:

```
This parameter of the operator `-` should be a number, a timestamp, a duration, a timeframe or an ip address, but was an array.

This parameter of the operator `-` should be a number, a timestamp, a duration or an ip address, but was an array.

This parameter of the operator `/` should be a number or a duration, but was an array.
```

These appear to be type validation error messages indicating that certain operators are receiving array values when they expect scalar values like numbers, timestamps, durations, timeframes, or IP addresses.