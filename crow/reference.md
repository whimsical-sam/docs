---
layout: page
permalink: /crow/reference/
---


# Reference

## input

`input` is a table representing the 2 inputs

### input queries

```lua
input[n].volts  -- gets the current value on input n
input[n].query  -- send input n's value to the host -> ^^stream(<channel>,<volts>)
```

### input modes

available modes are: `'none'`, `'stream'`, and `'change'`

```lua
input[n].mode = 'stream' -- set input n to stream with default time

input[n].mode( 'none' )         -- set input n to 'none' mode
input[n].mode( 'stream', time ) -- set input n to 'stream' every time seconds
input[n].mode( 'change', threshold, hysteresis, direction ) -- set input n to:
    -- 'change':   create an event each time the threshold is crossed
    -- threshold:  set the voltage around which to change state
    -- hysteresis: avoid noise of this size (in volts)
    -- direction:  'rising', 'falling', or 'both' transitions create events
```

table calling the input will set the mode with named parameters:

```lua
-- set input n to stream every 0.2 seconds
input[n]{ mode = 'stream'
        , time = 0.2
        }
```

the default behaviour is that the modes call a specific event, sending to the host:

```lua
'stream' -> ^^stream(<channel>,<volts>)
'change' -> ^^change(<channel>,<state>)
```

you can customize the event handlers:

```lua
input[1].stream = function(volts) <your_function> end
input[1].change = function(state) <your_function> end
```

## output

### slewing cv

```lua
output[n].slew  = 0.1 -- sets output n's slew time to 0.1 seconds.
output[n].volts = 2.0 -- tell output n to move toward 2.0 volts, over the slew time

v = output[n].volts   -- set v to the instantaneous voltage of output n
```

### actions

outputs can have `actions`, not just voltages and slew times.

```lua
output[n].action = lfo() -- set output n's action to be a default LFO
output[n]()              -- start the LFO

output[n]( lfo() )       -- shortcut to set the action and start immediately
```

available actions are (from `asllib.lua`):

```lua
lfo( time, level )             -- low frequency oscillator
pulse( time, level, polarity ) -- trigger / gate generator
ramp( time, skew, level )      -- triangle LFO with skew between sawtooth or ramp shapes
ar( attack, release, level )   -- attack-release envelope, retriggerable
adsr( attack, decay, sustain, release ) -- ADSR envelope
```

actions can take 'directives' to control them. the `adsr` action needs a `false` directive in order to enter the release phase:

```lua
output[1].action = adsr()
output[1]( true )  -- start attack phase and pause at sustain
output[1]( true )  -- re-start attack phase from the current location
output[1]( false ) -- enter release phase
```

## ASL

actions above are implemented using the `ASL` mini-language.

```lua
-- a basic triangle lfo

function lfo( time, level )
    return loop{ to(  level, time/2 )
               , to( -level, time/2 )
               }
end
```

everything is built on the primitive `to( destination, time )` which takes a destination and time pair, sending the output along a gradient. ASL is just some syntax to tie these short trips into a journey.

```lua
-- an ASL is composed of a sequence of 'to' calls in a table
myjourney = { to(1,1), to(2,2), to(3,3) }

-- often clearer as a vertical sequence
myjourney = { to(1,1)
            , to(2,2)
            , to(3,3)
            }

-- assign to an output and put it in motion
output[1]( myjourney )
```

ASL provides some constructs for doing musical things:

```lua
loop{ <asl> } -- when the sequence is complete, start again
lock{ <asl> } -- ignore all directives until the sequence is complete
held{ <asl> } -- freeze at end of sequence until a false or 'release' directive
times( count, { <asl> } ) -- repeat the sequence `count` times
```

### functions as arguments

ASL can take functions as arguments to get fresh values at runtime. this feature is essential if you want your parameters to update at runtime. this is how we get a new random value each time the output action is called:

`output[n]( to( function() return math.random(5) end, 1 ) )`

in this way you can capture all kinds of runtime behaviour, like a function that fetches the state of input 1:

`function() return input[1].volts end`

to aid this, a few common functions are automatically closured if using curly braces:

```lua
output[n]( to( math.random(5), 1 ) ) -- gives one random, but unchanging value
output[n]( to( math.random{5}, 1 ) )
                          ^^^ a new random value is calculated each time
```

this functionality is provided for:

```lua
math.random
math.min
math.max
```

## metro

crow has 7 metros that can be used directly:

```lua
-- start a timer that prints a number every second, counting up each time

metro[1].event = function(c) print(c) end
metro[1].time  = 1.0
metro[1]:start()
```

```lua
-- create a metro with the name 'mycounter' which calls 'count_event'

mycounter = metro.init{ event = count_event
                      , time  = 2.0
                      , count = -1 -- nb: -1 is 'forever'
                      }

function count_event(count)
    -- do something fun!
end

mycounter:start() -- begin mycounter
mycounter:stop()  -- stop mycounter
```

```lua
-- update params while the timer is running like so:

mycounter.time = 0.1
metro[1].event = a_different_function
```

## ii

```lua
ii.help()          -- prints a list of supported ii devices
ii.<device>.help() -- prints available functions for <device>
ii.pullup( state ) -- turns on (true) or off (false) the hardware i2c pullups
```

## cal

```lua
cal.test()    -- re-runs the CV calibration routine
cal.default() -- returns to default calibration values
cal.print()   -- prints the current calibration scalers for debugging
```

## crow

Accessed with `crow` or `_c`:

```lua
-- send a formatted message to host
crow.tell( name, <args> ) -> ^^name(arg1,arg2)

-- deactivate input modes, zero outputs and slews, and free all metros
crow.reset()
```

## globals

These will likely be deprecated / pulled into `_c` or their respective libs

```lua
time() -- returns a count of milliseconds since crow was turned on
get_out( channel ) -- send the current output voltage to host
get_cv( channel )  -- send the current input voltage to host
```
