---
author: Riley
---

## Part 0: Introduction

Real-time graphics, specifically in the context of game engines, is the
reason I got into programming. It should come as no surprise, then, that
most of the personal projects I have started have been related to graphics
or games in one way or another. However, if you were to look more closely
at one of these many repos, you would find, for the most part, abandoned
wastelands filled with incomplete features. Since I have begun seriously
attempting to learn graphics programming, I have left dozens of
half-complete renderers in my wake, some of which have done pretty neat
things, but most of which fail to get far beyond the initial "Hello,
triangle!" program.

Yet here I find myself, again starting a new project and again faced with
the overwhelming probability that it shall be abandoned. But this time is
different. This time, I have a purpose: to build a foundation from which to
learn.

In order to move on to more advanced topics in my computer graphics education,
I need to have a jumping-off point from which I can start new experiments.
The process of spinning up a new project from scratch is time-consuming and leads
to project burn-out. My goal now is to build that solid foundation on top of
which any number of more complex projects can be implemented.

By doing this, it is my hope that I can slowly transition
from my current level to someone with a more solid grasp of the field.
Whether that will actually happen remains to be seen. But nevertheless, I will
document my journey here, just in case anyone else finds it interesting. And to keep
me motivated.

The project, which I have dubbed the Lyman Graphics Rendering Engine (LYGRE),
is written in Rust and hosted online on [GitHub](https://github.com/rileylyman/lygre). 
The commit accompanying this post can be found [here](https://github.com/rileylyman/lygre/commit/5579f0a9f5a64cef3c89ab21da4eee85ee3487c2).
The commit is not documented or meant to be pretty, as it is really just a barebones
starting point for the project.

## Part I: The Bug Makes Itself Known

The focus of this blog will be more on (mis)adventures that are had during
the implementation of Lygre, rather than on the theory behind the
code being implemented. In that vein, this first post will focus on
a mishap that occurred just today, on the very first day of the project, and
will hopefully serve as useful to anyone new to debugging graphics programs.
(For the record, I am one of those people).

I sat down around noon, ready to crank out a large chunk of the
boilerplate I would need to implement in these first few weeks. I began with a few simple tasks.

_Step 1: Get a window to open up._ The [glfw](https://crates.io/crates/glfw) crate let me
accomplish this in just a few lines of code. I basically just cut and paste the code listed
on their documentation's homepage. Awesome.

(From the crate's documentation)
```rust
use glfw::{Action, Context, Key};

fn main() {
    let mut glfw = glfw::init(glfw::FAIL_ON_ERRORS).unwrap();

    let (mut window, events) = glfw.create_window(300, 300, "Hello this is window", glfw::WindowMode::Windowed)
        .expect("Failed to create GLFW window.");

    window.set_key_polling(true);
    window.make_current();

    while !window.should_close() {
        glfw.poll_events();
        for (_, event) in glfw::flush_messages(&events) {
            handle_window_event(&mut window, event);
        }
    }
}

fn handle_window_event(window: &mut glfw::Window, event: glfw::WindowEvent) {
    match event {
        glfw::WindowEvent::Key(Key::Escape, _, Action::Press, _) => {
            window.set_should_close(true)
        }
        _ => {}
    }
}
```

Step 2: Get OpenGL loaded and initialized. I linked in the [gl](https://crates.io/crates/gl)
crate and added a few lines from their documentation which worked perfectly.

```rust
// Added these lines after `glfw::init(...)`
glfw.window_hint(glfw::WindowHint::ContextVersion(4, 5));
glfw.window_hint(glfw::WindowHint::OpenGlProfile(
    glfw::OpenGlProfileHint::Core,
));

// Snip...

// Added this after `window.make_current()` to load the OpenGL function pointers.
gl::load_with(|s| window.get_proc_address(s));
```

Step 3: Get a triangle drawn to the screen. At this point, I was feeling pretty confident. I had
done this sort of setup dozens of times and was ready to breeze through it. For those who are
interested in a tutorial for this step, I have found [this](https://learnopengl.com/Getting-started/Hello-Triangle) 
to be a great resource. 

```rust
let triangle = vec![0.0f32, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 1.0, 0.0];

let mut vbo: gl::types::GLuint = 0;
let mut vao: gl::types::GLuint = 0;
unsafe {
    gl::GenVertexArrays(1, &mut vao);
    gl::GenBuffers(1, &mut vbo);

    gl::BindVertexArray(vao);

    gl::BindBuffer(gl::ARRAY_BUFFER, vbo);
    gl::BufferData(
        gl::ARRAY_BUFFER,
        triangle.len() as isize,
        std::mem::transmute(triangle.as_ptr()),
        gl::STATIC_DRAW,
    );

    gl::VertexAttribPointer(
        0,
        3,
        gl::FLOAT,
        gl::FALSE,
        (3 * std::mem::size_of::<f32>()) as i32,
        std::mem::transmute(&0)
    );
    gl::EnableVertexAttribArray(0);
}

// Snip shader compilation and linkage...

gl::ClearColor(0.5, 0.3, 0.7, 1.0);

while !window.should_close() {
    glfw.poll_events();
    for (_, event) in glfw::flush_messages(&events) {
        handle_window_event(&mut window, event);
    }

    unsafe {
        gl::Clear(gl::COLOR_BUFFER_BIT);

        gl::BindVertexArray(vao);
        gl::UseProgram(program);
        gl::DrawArrays(gl::TRIANGLES, 0, 3);
    }

    // This was missing at first:
    window.swap_buffers();
}

```


So, I generated my vertex array and my array buffer, buffered my data in 
and set up my attribute pointers, and made my draw call. And then...

Nothing. The screen was totally black. I took a few seconds to look back through my code: aha! I wasn't
swapping buffers. I add a call to `window.swap_buffers()` and... well, at least now I can see my
clear color being rendered. But still no triangle.

I looked back at my code. I checked that I was doing the API calls in the correct order. I began
inspecting all of the arguments to my OpenGL calls and came across this invocation:

```rust
gl::BufferData(
    gl::ARRAY_BUFFER,
    triangle.len() as isize,
    std::mem::transmute(triangle.as_ptr()),
    gl::STATIC_DRAW,
);
```

Oh, that was dumb of me. Looking at the documentation for `glBufferData` on [docs.gl](https://docs.gl/gl4/glBufferData)
reveals that I was specifying the `size` parameter wrong. The documentation reads
>Specifies the size in bytes of the buffer object's new data store.

The keyword there being _byte_ offset. Instead of writing `triangle.len()`, which was the word-size of the buffer,
I needed to specify `triangle.len() * sizeof(float)`. I changed the call to read:

```rust
gl::BufferData(
    gl::ARRAY_BUFFER,
    (triangle.len() * std::mem::size_of::<f32>()) as isize,
    std::mem::transmute(triangle.as_ptr()),
    gl::STATIC_DRAW,
);
```

And....... still nothing?? I had to take a step back at this point and think through my options.

There is a good presentation on graphics debugging (slides [here](https://docs.google.com/presentation/d/1LQUMIld4SGoQVthnhT1scoA3k4Sg0as14G4NeSiSgFU/edit?usp=sharing))
that enumerates a few things to try in a situation like this. On slide 8, the author lists:

1.) *Is your math right?* Well, I wasn't doing any math, and my triangle coordinate components were all either 0.0 or 1.0, so I was definitely
in screen-space.

2.) *Where's your camera?* Again, no camera here.

3.) *Is your geometry getting culled?* Hmm... maybe I was specifying my vertices in the wrong order and OpenGL was culling the triangle because
we were looking at its backface. I added a `gl::Disable(gl::CULLL_FACE)` call to double-check, but no cigar.

4.) *Are you clearing?* Yep, I was definitely clearing.

5.) *Are you actually drawing anything?* I double-checked my draw call. Everything was good there as well.

I went through the entire list and was still in the exact same spot. Getting a bit frustrated, I decided to pull out the big guns.

## Part II: Uncovering the Issue

[Renderdoc](https://renderdoc.org/docs/introduction.html) is a graphics debugger that allows you to disect every call you made
on a given frame of rendering. I had learned about it a few days early from [The Cherno's](https://www.youtube.com/channel/UCQ-W1KE9EYfdxhL6S4twUNw)
YouTube channel, but had never tried it out myself. With no where else to turn, I decided I would give it a shot.

I opened up Renderdoc for the first time and selected File>Launch Application, which brought me to this screen.

![Renderdoc]({{site.url}}{{site.baseurl}}/assets/images/210815/renderdoc-launch.png)

I input the path to my executable and launched the program, which opened up my application and printed some extra text on top of it:

![Renderdoc]({{site.url}}{{site.baseurl}}/assets/images/210815/renderdoc-capture.png)

Upon pressing F12, Renderdoc saved the current frame and I was able to inspect it in fine detail:

![Renderdoc]({{site.url}}{{site.baseurl}}/assets/images/210815/renderdoc-initial.png)

As you can see, the right-hand window shows the output of the current frame: a completely blank window. No triangle in sight.
On the left-hand side of the screen, a list of events that occurred that frame are shown. I am interested in why the `glDrawArrays` call is
not working, so I click on it to get a more detailed view.

![Renderdoc]({{site.url}}{{site.baseurl}}/assets/images/210815/renderdoc-input.png)

Wait a second... there's a triangle on the right-hand side of my screen now! If I look closer, I can see that Renderdoc is displaying
my input data to the current frame:

![Renderdoc]({{site.url}}{{site.baseurl}}/assets/images/210815/renderdoc-input-closer.png)

Okay, so the preview makes it seem like the triangle we are specifying is showing up at some point or another, but the window above is strange -- there
are no values for any of my input vertices, and all of the output vertices are completely 0. If I click on the output tab, I can see that this is
indeed the case: there is no output triangle in sight.

![Renderdoc]({{site.url}}{{site.baseurl}}/assets/images/210815/renderdoc-output.png)


At this point, I am still very confused. Just clicking around some more, I end up inspecting the call to `glBindVertexArray` to see what is going
on there.

![Renderdoc]({{site.url}}{{site.baseurl}}/assets/images/210815/renderdoc-vao.png)

There is not much info here, but I can see at the bottom of the screen a list of initialization parameters for the vertex array. 

![Renderdoc]({{site.url}}{{site.baseurl}}/assets/images/210815/renderdoc-vao-fields.png)

It looks like some of these correspond to values that I set, for instance, `glVertexAttribPointer` was set by me earlier on in the program.
I click on this parameter to expand it and make sure everything looks correct.

![Renderdoc]({{site.url}}{{site.baseurl}}/assets/images/210815/renderdoc-vao-uhoh.png)

Well, it looks like everything checks out... hold on. What is going on with the `offset` field? Why isn't it 0 like I expect?
I quickly realize that that long hex-string is a pointer to some
other location in my program's memory, and I must have made a mistake when casting some pointer value. Let's look back at my code:

```rust
gl::VertexAttribPointer(
    0,
    3,
    gl::FLOAT,
    gl::FALSE,
    (3 * std::mem::size_of::<f32>()) as i32,
    std::mem::transmute(&0),
);
```

Oh, dear. Looking at the last argument, it looks like I have made a very silly error. If you read through [the docs](https://docs.gl/gl4/glVertexAttribPointer),
you will find that the offset field, called `pointer`, has the following documentation:
>Specifies a offset of the first component of the first generic vertex attribute in the array in the data store of the buffer currently bound to the GL_ARRAY_BUFFER target. The initial value is 0.

The type of the value is `const GLvoid* pointer`, and so there are two ways to interpret this.
The first interpretation is that `pointer` _points_ to a memory location that contains the offset to use, as the name would imply.
This seems to be the interpretation I went with, since I passed OpenGL a pointer to the value of `0`, meaning that I thought
OpenGL would dereference my pointer and use `0` as the offset value. This, however, was the incorrect interpretation.

The second interpretation is that the value of the `pointer` field itself _is_ the offset value, and it is just a pointer
type for no reason. This is the correct interpretation, as it turns out. Therefore, you instead of passed a pointer to `0`,
in C you would just cast the value `0` to be a pointer like `(void*)0`. Because a pointer with the value of `0` is just a null-pointer,
we can rewrite out call as follows:

```rust
gl::VertexAttribPointer(
    0,
    3,
    gl::FLOAT,
    gl::FALSE,
    (3 * std::mem::size_of::<f32>()) as i32,
    std::ptr::null(),
);
```

Running our code with that modification solves the problem.

![Renderdoc]({{site.url}}{{site.baseurl}}/assets/images/210815/it-works.png)

## Part III: Conclusion

It is sometimes the case that the smallest bugs take the longest amount of time to track down. This is especially true when using an API like 
OpenGL with very little debug feedback available to you. While some would say that the moral of the story here is to always double-check
your function arguments, it is not always possible to pick out the issue amongst the forest of code around you. While this bug did not 
necessarily require using a graphics debugger like Renderdoc to solve, I probably would have spent hours more looking around for a solution if I had not
had it at my disposal.

Hopefully, this served as a nice introduction to Renderdoc for those of you who have never used it. I get the feeling that this will not be
the last time it helps me out of a situation like this while I implement Lygre.
