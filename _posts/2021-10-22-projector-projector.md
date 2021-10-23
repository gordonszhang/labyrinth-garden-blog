---
layout: post
title: "Projector, projector on the wall"
tags:
  - tutorial
---

...Who's the most scuffed gamedev of them all?
Don't answer that. :)

## A blast from the past
Labyrinth Garden's plot has Lily taking various missions from the Mage's Guild to
investigate magic-related crime in August. Obviously, every good mission needs an 
equally good briefing. 

As such, I wanted to present mission briefings in a fun and lore-consistent
manner. And what better way than to use a slideshow to show the player what they're going up
against?

![](/assets/images/2021-10-22-projector-projector/Canvas.png) 

Because the game takes place in a city that's just starting to adopt modern
Ether-powered technology, I took inspiration from old slide
projectors, which use colored slides and a bulb to project images.

## The how-to part
With the idea out of the way, I quickly set to work implementing a slide projector
canvas in my game.

I knew I needed to accomplish the following things to make the effect work:

1. Animate a projector canvas opening and closing
2. Animate slides being swapped
3. Expose functions like opening, closing, and toggling the picture of the projector
  so that they can be called during cutscenes

### Part 1. Canvas opening and closing
I'm lazy, so rather than animating a sprite of the canvas opening and closing,
I decided that I could save some effort by using masks. Here's how:

First, I drew out a canvas in my pixel art tool of choice, Aseprite.
I also use [Aseprite2Unity](https://seanba.itch.io/aseprite2unity) to streamline
importing of animations and sprites so I can use them in the editor
without having to slice spritesheets or re-slice when I make changes.

![](/assets/images/2021-10-22-projector-projector/ProjectorCanvas.png)

Then I laid out a sprite mask and a "bottom bar" sprite to represent the bottom
"spool" of the canvas as it rolls out.

`CanvasMask`             |  `BottomBar` 
:-------------------------:|:-------------------------:
![](/assets/images/2021-10-22-projector-projector/ProjectorCanvasMask.png)  |  ![](/assets/images/2021-10-22-projector-projector/BottomBar.png)

In Unity, I set up some empty GameObjects with this structure:
![](/assets/images/2021-10-22-projector-projector/Part1Unity.png)

I put the canvas sprite on `Canvas`. 
![](/assets/images/2021-10-22-projector-projector/CanvasUnitySetup.png)
Note that Mask Interaction is set to `Visible Inside Mask`
and the Sorting Layer is `BG1`. That'll be important in a moment.

For the `BottomBar`, I set the sprite and added a Sprite Mask. 
![](/assets/images/2021-10-22-projector-projector/BottomBarUnitySetup.png)
For the Sprite
Mask, I used the `CanvasMask` created above, which is big enough to mask the
entire canvas. And the range is set to only effect a subset of the `BG0` sorting
layer. This is necessary since I'll be using masks for the slide swap effect.

The final effect is that when the `BottomBar` is moved up and down, the mask
moves with it, effectively dictating how much of the canvas is visible behind it.
This gives the illusion of the canvas rolling in and out!
![](/assets/images/2021-10-22-projector-projector/projector_mask.gif) 

### Part 2. Showing slides and swapping animation

For the actual slides, I need to be able to overlay sprites on the canvas,
mask out the edges, and animate slides swapping out.

Back to Aseprite:

`ProjectorSlides`             |  `ProjectorSlideMask` 
:-------------------------:|:-------------------------:
![](/assets/images/2021-10-22-projector-projector/ProjectorSlides.png)   |  ![](/assets/images/2021-10-22-projector-projector/ProjectorSlideMask.png) 

Note that the `ProjectorSlides` file has two frames, each with a numbered tag.
At a high level, the ProjectorCanvas will keep track of the slide `index`.
When we want to move to the next slide, we increment the index and get the
sprite with the associated frame tag.

The `ProjectorSlideMask` will serve to mask the ProjectorSlides, so we can
move the slide image to the side without it popping out of the canvas.

Back to Unity! We add GameObjects for the slides, the mask and a light.
![](/assets/images/2021-10-22-projector-projector/Part2Unity.png)

I add another Sprite Mask using the `ProjectorSlideMask` sprite above, and
set it to target a specific range of the sorting layer `BG1`. This will ensure
that it doesn't mask the canvas sprite itself.
![](/assets/images/2021-10-22-projector-projector/SlideMaskUnitySetup.png)

For the slides, I add a SpriteRenderer and an Animator, which just contains
the two animation states `0` and `1` for each slide. This could be extended
for more slides.
![](/assets/images/2021-10-22-projector-projector/SlidesUnitySetup.png)

The mask will let us slide the slides in and out as desired.
![](/assets/images/2021-10-22-projector-projector/slides_mask.gif)

Lighting for my game is handled by the experimental [Lights 2D](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@7.1/manual/Lights-2D-intro.html) Unity package. I added a light to the canvas which will
give it the effect of appearing illuminated.

![](/assets/images/2021-10-22-projector-projector/LightUnitySetup.png)

### Part 3. Driving the projector with code
Now we're getting somewhere!

For those who want it, I've just pasted the whole class below at the bottom of the post.
I use a tweening library, DOTween to handle the animation of objects, like
moving around and toggling various variables.

Swapping slides out is a combination of tweening the slide SpriteRenderer's 
x position and telling the slide deck Animator to play the animation for the
next slide.

```
DOTween.Sequence()
    // tween slide to the left
    .Append(SlideRenderer.transform.DOLocalMoveX(-56, 0.3f, true).SetEase(Ease.OutQuad))
    .AppendCallback(() => {
        // increment slide index
        _slideIndex++;                          
        // now play that animation
        SlideAnimator.Play($"{_slideIndex}");   
    })
    // wait
    .AppendInterval(1f)                         
    // tween slide back to the right
    .Append(SlideRenderer.transform.DOLocalMoveX(0, 0.3f, true).SetEase(Ease.OutQuad));
```

To toggle lights, I set the intensity of the light.
```
PointLight.intensity = 0f;   // zero for "off"
PointLight.intensity = 0.7f; // non-zero value for "on"
```

Finally, I pass Tweens as a return value so that my custom cutscene handler
can block on the animations until they finish before moving on.

![](/assets/images/2021-10-22-projector-projector/example_cutscene.png)

And the effect in action:

![](/assets/images/2021-10-22-projector-projector/projector.gif)

Thanks for reading this post, and hopefully you found it helpful!

```
using UnityEngine;
using UnityEngine.Experimental.Rendering.Universal;
using DG.Tweening;

public class ProjectorCanvas : MonoBehaviour, IResettable {
    public Light2D PointLight;
    public SpriteRenderer CanvasRenderer;
    public SpriteRenderer SlideRenderer;
    public Animator SlideAnimator;
    public Transform BottomBar;
    private int _slideIndex;

    public void Start() {
        ResetObject();
    }

    public void ResetObject() {
        TurnOff();
        BottomBar.transform.localPosition = new Vector2(0, SlideRenderer.size.y);
    }

    public Tween Open() {
        return DOTween.Sequence()
            .Append(BottomBar.DOLocalMoveY(0, 2f, true))
            .SetEase(Ease.Unset);
    }

    public void TurnOn() {
        PointLight.intensity = 0.7f;
        SlideAnimator.Play($"{_slideIndex}");
        SlideRenderer.enabled = true;
    }

    public Tween NextSlide() {
        return DOTween.Sequence()
            .Append(SlideRenderer.transform.DOLocalMoveX(-56, 0.3f, true).SetEase(Ease.OutQuad))
            .AppendCallback(() => {
                _slideIndex++;
                SlideAnimator.Play($"{_slideIndex}");
            })
            .AppendInterval(1f)
            .Append(SlideRenderer.transform.DOLocalMoveX(0, 0.3f, true).SetEase(Ease.OutQuad));
    }

    public Tween Close() {
        return DOTween.Sequence()
            .AppendCallback(() => {
                TurnOff();
            })
            .Append(BottomBar.DOLocalMoveY(SlideRenderer.size.y, 2f, true))
            .SetEase(Ease.Unset);
    }

    public void TurnOff() {
        SlideRenderer.enabled = false;
        PointLight.intensity = 0;
        _slideIndex = 0;
    }
}
```
