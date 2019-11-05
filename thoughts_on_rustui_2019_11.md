# Thoughts on Rust UI
### 2019 November

I have caught, off-and-on, a lot of talk about UI in Rust over the last couple of years. Lots of people have had lots of ideas, but overall the message seems to be that...
1. UI is hard.
2. We don't know the best patterns for UI in Rust.
    * Many (most?) UI paradigms are object-oriented. Rust is not.
    * There are lots of experiments along various vectors.
3. Rust would be a great language for UI (nevermind #2?).
4. More people would use more Rust if there were a good UI solution.

I'm sure I missed some threads, mostly I'm just trying to get past some of the more typical talking points I see.

There are some thoughts I have that I don't see brought up (ever? often?) which I think we should consider when talking about user interfaces and Rust. I come at this as a professional developer who uses C++ and `Qt` primarily; but also having had experience with `wxWidgets` and early-era Java/Swing, Windows, and MacOS 9/X UI work while I was learning programming.

For Rust UI tools, I've used mostly `imgui-rs`, but I've also done at least the tutorials for `Azul` and I've recently been playing with `Cursive`. But I've _tried_ to do some basic things in a few others, and read through the intro docs and basic examples for most of the major toolkits I've seen mentioned. And I have thoughts.

---

## tl;dr

This is a long-ass article. Sorry. So here are (IMO) the major takeaway goals that I would have for Rust UI -- as a professional UI developer -- that I think we need to be working towards:

1. UI and app state are owned by the same thing so you don't have to be `clone()`-ing wrappers inside your callbacks, nor using the `lazy_static!` crate for the definitive "Rust UI Program" model.
2. A complete UI toolkit needs to have robust widgets that look and feel good on a desktop.
3. We should minimize "magic" helpers (meaning hidden functions and templates that don't look like Rust). Learning any library is harder and more frustrating when you have to learn its secret callbacks before you can do anything with it.
4. It's fine (I might argue desirable) to closely associate structs with callbacks to draw their UI.

I don't really get into actual event callback paradigms below, but to an extent I think that the above requirements sort of take care of that on their own. That is, closures can be really aggravating, but if your framework doesn't require `clone()`s of the app state or weird magic macros, then closures or any other function callbacks should be fine.

The broad goal for me is that an "idiomatic Rust GUI" should look and smell like "idiomatic Rust."

---

Okay, now for those who want to dig in to more detail than anyone asked for, you may now proceed below.

## What is a UI program?

Data ownership being one of the best featured of Rust is also going to be vital to address in any UI system. In a typical object-oriented approach, I would generally have a global repository of "app data" which can be accessed as needed. Most UIs run on a single thread, so it's pretty safe to do so, but if you're worried, you can put all your data queries into scoped mutex containers or something, and it's pretty stable.

But essentially, all your _data_ is global. And we tend to not like that in Rust. I feel like data ownership is probably one of the biggest hurdles to deal with here.

And it's not super simple, either, because any UI system also must own its own data about positioning of widgets and extra display information (label text, colors, menu options, and event callbacks). Those data aren't really related to core app data except that they will often mirror some elements, and they need to be updated when _actual_ app data changes. So let's look at some approaches.

## 1. `lazy_static!`

I haven't messed around too much with this one, but basically, because mutating static data is inherently `unsafe`, Rust doesn't normally let you create singletons or some equivalent of static classes that can be updated. `lazy_static!` allows you to create static variables which require runtime and/or heap initialization. Using `lazy_static!` you could create your "static class" or "singleton", and you could then proceed to use this as you would normally.

There's actually a much better explanation of it over at [this StackExchange answer](https://stackoverflow.com/questions/27791532/how-do-i-create-a-global-mutable-singleton), but effectively you can use `lazy_static!` if you absolutely need to, but you should think long and hard about whether or not you actually need to.

I would argue that UI app state is a VERY solid situation where it's appropriate to use mutable static data; but it's hard to find anyone talking about whether or not they agree with me.

So let's assume I'm wrong and we don't want to use statics (also, `lazy_static!` isn't part of the core language, and if we should always be striving to reduce our reliance on unsafe code). The next option might be...

## 2. Rc/Arc Cloning

If we can't have mutable static data, then we need to make our state structure before we do any UI work, and then we pass things around.

The basic loop around any UI setup is that you do some amount of initial setup, hook up some pieces (more or less, depending on the UI system), and then execute a `run(...)` loop on the base UI struct, which then handles all of its data and (when appropriate) calls into functions and/or data that you have bound to the system.

Here is a snipped from a CLI farming game I've been playing with using Cursive:
```rust
pub fn bootstrap_farm() {
    let game = Rc::new(RefCell::new(FarmGame::new()));
    let mut ui = Cursive::default();
    ui.add_global_callback(Key::F4, |ui| ui.quit());

    // ...

    // Show new farm
    setup_new_farm_ui(&mut ui, game);

    ui.run();
}
```
I set up my app state (in this case, the `game` object), create a new Cursive `ui` object, pass that around to some new UI functions, and then once the initial setup is built, call `ui.run()` to begin Cursive's main loop. The app flow is now basically out of my hands. All of my callbacks have either been set up; and within those callbacks I might change modes based on user interaction, and those callbacks will then change the UI and give it _new_ callbacks.

All of these callbacks need to access the game state, though, which explains why my game state is the infamous `Rc<RefCell<T>>` type that we (I assume) love so much in Rust. It's (more-or-less) the `shared_ptr<T>` from C++, and the... basically all objects from C#.

Using the `Rc<Refcell<T>>`, I can pass immutable copies of my reference-counted pointer to the internally-mutable data I made. I can then use the `Rc<RefCell<T>>` to get a mutable reference to my game state whenever I need to run some operation or update the UI.

But this makes to rather ugly code. Consider this setup function:
```rust
fn setup_new_farm(ui: &mut Cursive, game: Rc<RefCell<FarmGame>>) {
    let game = game.clone();
    ui.add_layer(Dialog::text("No farms found.").title("Farm Game").button(
        "New Farm",
        move |ui| {
            setup_name_player_view(ui, game.clone());
        },
    ));
}
```
The first thing I do is `clone()` the `Rc<RefCell<FarmGame>>` so that I can move it into the closure captured on the "New Farm" button. Inside the closure, it's then cloned _again_ when called because you can't move the `Rc<RefCell<>>` because this closure could be called again.

Looking at my code thus far, basically any time I have a callback which changes the UI, I've got two clones going on, and to be frank, that seems pretty excessive.

Also (and I read the reasoning before, and it made sense at the time), Rc is necessarily not `Copy`, meaning that you can't just pass it around like you could do with a `shared_ptr<>` in C++. So literally any time you want to use it, you have to make sure you're working with a copy you explicitly intended to make; and because likely the vast majority of your UI will need to be interacting with the state of your program, you're going to be performing this `clone()`-fu everywhere.

As I said before, that seems excessive. But what if we could just... pass around a mutable reference in a way that means we don't need to be cloning pointers nonstop...?

## 3. Immediate-Mode GUI

Immediate-mode GUI seems to be a relatively "natural" feeling UI system for Rust (meaning that I don't feel like I'm forced into doing coding backflips just to get my data safely to where it needs to be for the UI to use it). The data is still owned by the main function, but it no longer requires excessive cloning of any containers.

The immediate-mode GUI setup begins very similarly to Cursive:
```rust
fn main() {
    let mut state = AppState::default();
    support_glium::run(
        "Monster Hunting Wishlist+".to_owned(),
        CLEAR_COLOR,
        |ui, _, _| {
            state.layout(&ui);
            state.process_events();
            !state.should_quit()
        },
    );
}
```
You may note that I don't need to use any `Rc<RefCell<T>>` objects here. The `state` data is passed in to the main UI loop and basically all my functions for laying out the data are passed a (mutable) reference. Because the UI runs on a single thread, there is never more than a single reference which is passed in and then comes back out of helper functions.

This kind of sounds ideal, except that (typically) with immediate-mode UIs you are redrawing the entire UI every frame -- which means that no matter how quickly your program goes through the process, it's likely going to be doing that 60 times per second.

So okay, let's talk real-world though. `imgui` is pretty well optimized, and I believe it does a fair amount of caching under the hood, so if your UI doesn't change, it's not going to be regenerating display lists every frame. There's some savings there, to be sure. And even more so, you could probably fairly reasonably optimize a pure-UI app to only run the update loop when there are UI events to process, so more like between 0 and 60 time per second.

Even with these optimizations, you're still going to have a pretty hefty UI loop every time it goes through, which is really unfortunate. I don't want my UI programs to eat up hardly _any_ CPU time when they're literally just sitting there doing nothing.

So what if we found some kind of compromize between immediate-mode and retained mode? One that has all the caching of the UI elements, but also only considers regenerating the UI if the underlying data actually changes...?

## 4. UI owns the data

That's basically what frameworks like `Azul` are trying to accomplish. In `Azul`, you essentially give your app's state to the UI system on creation, and then it asks you to update the UI whenever it deems it necessary to do so; and in your callbacks, you don't have to use closures, but instead you accept an argument a wrapper around your data model so that `Azul` can pass you a reference as needed when the user interacts with the UI.

Here is (most of) `Azul`'s "Hello World" example from the homepage:
```rust
struct DataModel {
    counter: usize,
}

impl Layout for DataModel {
    fn layout(&self, _info: LayoutInfo<Self>) -> Dom<Self> {
        let label = Label::new(format!("{}", self.counter)).dom();
        let button = Button::with_label("Update counter").dom()
            .with_callback(On::MouseUp, Callback(update_counter));

        Dom::div()
            .with_child(label)
            .with_child(button)
    }
}

fn update_counter(event: CallbackInfo<DataModel>) -> UpdateScreen {
    event.state.data.counter += 1;
    Redraw
}

fn main() {
    let mut app = App::new(DataModel { counter: 0 }, AppConfig::default()).unwrap();
    let window = app.create_window(WindowCreateOptions::default(), css::native()).unwrap();
    app.run(window).unwrap();
}
```

You can see that the `app` binding is created using the custom-defined `DataModel` struct. Here's the `App::new` function signature from `Azul`'s code:

```rust
pub fn new(initial_data: T, app_config: AppConfig) -> Result<Self, WindowCreateError>
```

The whole `App` struct is a generic data structure built to safely care for your app state data. This is another one that feels pretty nice in Rust. The DOM syntax and setup for the various layout functions can become a bit complex, but overall it works well. That is, if your program is UI-driven, you may as well let the UI system manage your data to avoid multiple mutable reference errors and excessive `Rc`-cloning.

It's also fairly compelling being able to create custom layouts for every sub-struct within your app state -- it feels nice and compartmentalized, and (IMO) has a pretty good proximity between the UI code and the data it's meant to be displaying. It's almost like you create whole custom widgets for each data type and lay them out! In fact, to an extent that's sort of HOW you create new widgets if you want to.

Yet there lies another problem with the Rust UI ecosystem: non-native UI toolkits need to be fleshed out. The last time I used `Azul` (admittedly several months ago now), it didn't really have very many widgets, and the widgets it DID have were little more than the laid-out rectangles of a WebRender view with some CSS applied. Labels and buttons all worked, but they didn't come with any normal UI feedback like highlights when the cursor hovers over a button, nor does the button change to a sunken-in look (or simply a darker color) when you press it. There are no menus or spinners or anything yet.

I don't mean this as a dig on `Azul` -- I actually really like what it's doing -- but the readme itself says that it's not ready yet, and until it gets more (and better) widgets it just won't be ready. Custom UI toolkits need to be fleshed out with responsive, decent-looking widgets that might look like they make sense on a desktop app; and that takes time and likely a lot of community support.

> It's worth noting that because `Azul` supports SVG rendering, one could theoretically create resizable buttons and backgrounds and any other needed widget images (e.g., if regular CSS coloring is insufficient). I am a bit saddened that this doesn't appear to have been done already.

### OrbTk

`OrbTk` seems to be another where the app's data is owned by the UI. It claims React as inspiration, which is great and I really like React's paradigm. It looks great, the examples seem to make sense (though all of that indentation seems like it'll be really rough in most of the more involved examples).

But the last time I ran it, the calculator example was painfully slow; and it seems like it's been in a "don't bother using it, we're working on 0.3.0 which will change a lot" mode for pretty much the whole year.

## Toolkits I didn't mention

So there are a lot of toolkits I didn't mention, mostly because I either don't know them well or I feel like they have one or more of the problems I mentioned above. Let's just review them real quick...

#### Gtk-rs

Not only is this toolkit fairly inscrutible for people new to Gtk, it also has the same "who owns the data" problem as #1 and #2 above. You either need a static or to pass around cloning -- only worse, you also end up needing to clone the UI widget references themselves in order to... convert them... into other types? In order to do things to them?

Look, there are lots of weird things which, I'm sure, make more sense in C with Gtk, but the system is object-oriented and my humble opinion is that it doesn't work well in Rust, though the devs HAVE done a pretty stellar job of binding the whole thing so it works at all.

And, last and least, I do all my dev on Windows and Mac, and while you _can_ get Gtk running on both, it's a pain and the resulting programs almost always feel wrong without a whole lot of extra effort and increased binary distribution size. Also, the GPL might play into this at some point, which means that a commercial product could have problems building off of Gtk.

#### Relm

Being based on `Gtk-rs`, this one sort of suffers from the same problem where you need to already know Gtk in order to use it properly. I tried using Relm for a day or two, but ultimately it seemed more natural/faster for me to just ditch it altogether and use pure-`Gtk-rs`.

`Relm` also makes pretty heavy use of their `view!` macro, which it a level of "magic" that I really prefer to stear clear of when dealing with libraries (and then I tried it, I found that I quickly needed feature that it either didn't offer or I couldn't figure out because it's a hidden feature that wasn't explained obviously enough or in enough detail to figure out how to make it work for me).

#### IUI

`iui` is built atop `libui`, which while it has lots of widgets, it is also pretty up-front about _not_ having a lot of widgets. So ultimately it's the same problem as `Azul`; and there's also a conern (I haven't checked) that widgets laid out using code on one platform may not look right using the same code compiled to another platform. (If that's the case, you almost may as well write a whole UI crate for each native platform.)

It's actually less compelling for me to try using `iui` because if I decide I need a widget that hasn't been implemented yet, I actually need to then add it to the C `libui` library (make a fork of that) and then add the bindings for my new widget to `iui`, and then also (if I'm being responsible) implement that widget on the other supported platforms. And if I need to write a new widget, I would _much_ prefer a system where I can write it once and it works the same everywhere.

#### Iced

I haven't tried it, but it looks interesting. It seems to invert the "data is owned by UI" paradigm such that "UI is owned by data" via implementing necessary traits on your app data type.

It hides a very basic `run()` function from you that you have to just know about and call to get your UI started, but otherwise there's surprisingly little "magic" going on, and I like that. Though it, again, needs people to make more widgets and (if I'm being honest) needs to look more like a desktop UI. It looks a LOT like ImGui, which reads to me heavily like an in-game debug UI more than desktop-class application for Business People Doing Business.

#### Conrod

It seems like a great library with lots of widgets, but looking at the examples is _very_ intimidating. You're basically manually setting up a lightweight game-style render/intup loop for your UI; and I get it, that's kind of what all UI programs are on some level, but people looking to extend their CLI tool with a more friendly GUI probably shouldn't need to set up an entire render loop to do so.

Also, the widgets look very "programmer art" which is fine for the sake of fleshing out an idea and "leaving it to the community" to make it look better; but it won't be ready for production until there's a more professional looking theme available by default.

Given Conrod's game engine origins, it seems about as useful to use it for a larger application as it would be to use Unity to build a non-game UI application. Like, you totally could, and it would run fine; but you'd get a lot of funny looks, and you are probably also going to pay for extra CPU cost of rendering and file size/RAM cost around the fact that it's also a game engine running.

#### Various Other Bindings

Rust has some crates for Qt bindings, and I've heard that maybe `sciter-rs` might work okay. But I think many people (and several GUI repo readme files) have already expressed that C/C++/etc. bindings just doin't work well with Rust's type safe paradigms and they feel "forced" as I said way above.

## Closing Remarks

I think we have a lot of good progress so far -- there are a ton of excellent ideas in the works. I don't mean to single any crates out or pick on them (if I was extra critical of a crate it's likely because it was good enough for me to keep using longer to learn more about, which means I liked it better than most of the rest I've tried). Everyone has done a lot of good work.

I think a few ideas have bubbled to the surface in my mind about how to do GUI in Rust, and those can be found in the tl;dr section up at the top.

Pleast let me know if I've messed something up or misrepresented a toolkit here, or if you think I've missed a major piece of the Rust GUI scene that deserves a closer look!

We're at the point now where I think we need to get more hands on board some of these projects and really push them to get them to look and feel more "complete" both in terms of widget functionality and professional appearance. I encourage all of these crate owners to put together lists of tasks they want people to help out with. Offer some suggestions about how to flesh out more widgets for your system, or how might someone go about wiring their data model into the UI loop internally so that it can be included with all of the callbacks from buttons and events? Let us know how we can help you, please! I feel like if we band together, 2020 really could be the year we start to break through the UI ceiling into having Rust usable to build desktop GUI apps and tools!
