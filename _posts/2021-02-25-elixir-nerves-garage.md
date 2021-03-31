---
layout: post
title: Controlling my garage door with Elixir and Nerves
lead: Is the garage door open? Because my eyeballs can't answer this question in my house I explore using some  hardware and software to answer this question.
---

I've wanted to learn Elixir for years now and have been looking for a little project to explore with. The best way to learn a new language for me is to solve a concrete problem I have. Toy apps don't work for me.

I've also been following the Nerves project and am really interested in using Elixir/Erlang OTP model for embedded projects. Elixr/Erlang's *expect failures and deal with them* mantra really fits well with the embedded world. I've dabbled with embedded development for a long time and have always preferred to reach for C. It's super lightweight and tiny, but also super easy to screw up. With the availability and low cost of more powerful systems like Raspberry Pi Zero, you can more easily afford to move up to a higher-level language like Elixir without making too many trade-offs (yes I know MicroPython exists).

Automating my garage door seems like the perfect excuse to finally do something with Elixir and Nerves. I can't see my garage door from inside my house and I'm always wondering if it's open or closed. I could buy a $50 retail sensor/switch to do this, but that's not fun and it won't help me learn Elixir. I've also used a homebridge setup to do this in the past, but the homebridge ecosystem is maddening. Any given plugin has at least a dozen variants and finding the one that's been updated in the past decade is a chore.

## Hardware

- Raspberry Pi Zero W (the one with wireless networking)
- 5-volt relay module
- Magnetic reed switch

## Software

- Elixir and Nerves for firmware
- Home Assistant
- MQTT server (via Home Assistant)

## Firmware

<!--<img src="/images/door_sensor.jpg" style="height: 400px; float: right; margin: 1em 0 3em 3em;">-->


Let's start with the sensor. Something that will tell me if the garage door is open or closed. The simple way to do this is with a magnetic reed switch. It's a switch that has two contacts and a long thin metal reed and opens and closes the switch in the presence of a strong magnet. I mounted the sensor on the frame of the garage door and the magnet onto the door itself.

<img src="/images/door_sensor.jpg" style="height: 400px;">

The one I'm using has screw terminals for NC (normally closed) and NO (normally open) configurations. I'm using it in NO mode so the system will fail open. That way if the wires become disconnected for any reason, it will indicate that the door is open. What I don't want is for something to fail and indicate the the door is closed when it isn't.

Nerves has `CircuitsGPIO` for interacting with your board's general purpose input/output pins; that's what we'll use here. Install it by adding it to your mix dependencies:

```elixir
  {:circuits_gpio, "~> 0.4"}
```

I've created an Elixir module called `Garage.DoorSensor` that implements the `GenServer` callbacks required for this. The GenServer abstraction is always the piece I've struggled with in Elixir. Here's the minimum setup needed to sense the door state. It's all GenServer callbacks:

```elixir
defmodule Garage.DoorSensor do
  use GenServer
  require Logger

  @sensor_pin 26

  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init(_) do
    {:ok, gpio} = Circuits.GPIO.open(@sensor_pin, :input, pull_mode: :pullup)
    Circuits.GPIO.set_interrupts(gpio, :both)
    {:ok, %{gpio: gpio}}
  end

  def handle_info({:circuits_gpio, @sensor_pin, _timestamp, value}, state) do
    Logger.info("Garage door is #{value}")
    {:noreply, state}
  end
end
```

The `init` funciton is called when the `GenServer` process starts. It sets up the pin our sensor is connected to and configures an interrupt (a function that will be called when the pin state changes) to handle the state changes. What isn't super clear here for Elixir/Nerves newcomers is that the interrupt is a `GenServer` `info` message. This isn't explicitly documented anywhere. I had to look at the CircuitsGPIO examples and search though other Github repositories to sort this out.

I've connected to two leads of the sensor to pin 26 and ground. When the switch/door is open, pin 26 is connected to ground.

## Activating the opener

Controlling the door opener relay is a lot simpler—it's just a function call (`Circuits.GPIO.write`). I did wrap this up into a `GenServer` module in order to store the GPIO reference:

```elixir
defmodule Garage.DoorSwitch do
  use GenServer
  require Logger

  @switch_activation_time_in_ms 500
  @switch_pin 4

  def activate do
    GenServer.cast(__MODULE__, :activate)
  end

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def init(_state) do
    Logger.info("Door switch init...")
    {:ok, gpio} = Circuits.GPIO.open(@switch_pin, :output)
    {:ok, %{gpio: gpio}}
  end

  def handle_cast(:activate, state = %{gpio: gpio}) do
    Logger.info("Activating door switch...")
    Circuits.GPIO.write(gpio, 1)
    Process.sleep(@switch_activation_time_in_ms)
    Circuits.GPIO.write(gpio, 0)

    {:noreply, state}
  end
end
```

There's also the on/off timing logic. We can't just turn the relay on—that'd be like mashing down the opener button and never letting go. We want to simulate a button press. I chose 500ms (half a second) and it seems to work fine.

There are more `GenServer` callbacks here. Just like in `Garage.DoorSensor`, we have `start_link` and `init`. In this module we're using `GenServer.cast` because we don't need a response. It's just call-and-forget.

The other difference here is that we've added a public API—the `activate` function. This is the `GenServer` pattern you'll encounter: define public API functions and implement them in `GenServer` callbacks using the appropriate callback flavor (`call`, `cast`, and `info`).

## Tying it all together

The other bit of `GenServer` that has confused me in the past is how to start these processes up. My poor old programmer brain sees the world in OO. I've been thinking of these Elixir processes as object instances. You "instantiate" them and send them messages, right? So where do these things get intstantiated? The supervision tree. Your generated Nerves project skeleton will come an application file (`Garage.Application` in my case). This is where these child processes are defined and configured. Here's an abbreviated version of what I'm using:

```elixir
defmodule Garage.Application do
  use Application

  def start(_type, _args) do
    opts = [strategy: :one_for_one, name: Garage.Supervisor]

    children =
      [
        {Garage.DoorSensor, []},
        {Garage.DoorSwitch, []}
      ] ++ children(target())

    Supervisor.start_link(children, opts)
  end
end
```


<!--
## Outline

- Motivation
  - Want to learn Elixir and Nerves
- Problem
  - I want to know if my garage door is open (I can't see if from inside the house).
  - I can spend $75 on a retail sensor/switch. But that is not fun and I won't learn anything.
  - Bonus: activate garage door, sense service door (while we're at it)
- Hardware
  - Raspberry Pi Zero W
  - 5-volt relay module
  - Magnetic reed switch(es)
- Software
  - Elixir/Nerves firmware
  - Home Assistant
  - MQTT server (via Home Assistant)
- Firmware design
  - Sensor:
  - Here's where I'm revealing my noob-osity noob-aciousness: I'm not sure what the idomatic way to do this is.
    - Set up Circuits GPIO interrupt to detect door sensor state changes (rising and falling)
      - GenServer confusion: docs were unclear
      - image of rising, falling edges?
    - Configure sensor in NC mode. Either will work, but we want to to fail open.
    - Connect sensor to GPIO pin and ground. I chose a GPIO right next to a ground for convenience
    - Code snippet of logger output.
  - Relay:
    - Describe relay, graphic, why used here to activate opener
    - The relay is simpler: just tickle a function to toggle the state pin.
    - Code snippet
- Putting it all together with MQTT and Home Assistant
  - I love homekit. All local network, highly integrated UI, Siri...
  - Home Assistant supports everything, can expose HomeKit devices.
  - install
    - home assistant OS
    - MQTT server
      - TLS
  - Use Tortoise to send/receive MQTT messages
    - Talk about confusing docs
  - Switch/sensor setup in HASS  
- Next steps
  - Add another sensor to top of garage door to detect closed state and open state
  - rpi3 with POE, ethernet (wifi is flaky)
-->
