---
title: "Finding Unity Bugs While Profiling Memory"
date: 2023-05-31T00:00:00-00:00
# lastmod: 2023-05-31T00:00:00-00:00
tags: ['unity', 'profiling']
description: "Humans program bugs. Humans create your commercial game engine. Sometimes you stumble upon those bugs."
draft: false
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
---
Is it wrong to be suspicious of software provided to you by others? Of course not! Humans program bugs, and you use a lot 
of software written (primarily) by humans. So maintaining a healthy suspicion of the software we rely on is wise. 
However, when we discover a problem, that does not mean rushing to blame the software of others. Simply, we should 
approach an issue without bias and treat it as a fact-finding mission.

I can think of two occasions in the last few years where I discovered a bug in Unity's UGUI C# package. Both bugs could 
have been more exciting and complex to fix, and somebody already corrected the bugs in a newer version of Unity. I'm 
writing about the bugs because I stumbled upon the bug and identified the root cause.

Both bugs were similar. Some code in a C# class designed to cache references to objects would not drop 
references at the appropriate moment. The problem is that those cached references may be referencing UI textures. The 
class is responsible for managing the cache is reachable from a static root. A static root is something that is not going
to be garbage collected. A typical example of a static root is a singleton instance stored in a `private static` class 
field. Unity cannot unload the UI textures while there is a path back to a static root. Textures that may no longer be 
in use will be kept in memory.

I recall the first bug from memory, which was in `Selectable.cs`. Any active selectable UI element is cached and 
later dropped from the cache when it is no longer active or has been destroyed. Internally, the class maintained an 
array of objects. When the cache drops an object, its index in the array is replaced by the last populated 
element of the array. If I recall correctly, the bug was a simple logic bug in the swap code that resulted in dropped 
objects being left behind near the end of the array. The bug could self-correct as the array fills up with
references, which would overwrite the incorrectly left behind objects. The problem is that if the game goes from 
a scene with many cached objects to a scene with very few, then that scene will likely use more memory than required.

The second bug is more recent and very similar. This time the bug was in `Graphic.cs`. This class caches all active 
graphic components. It maintains a mapping to every active canvas and all its graphic components. An edge case could 
result in a graphic component being destroyed but not removed from the cache. I want to walk through a typical memory 
profiling session that leads to finding these bugs.

I stumbled upon the bug while analyzing textures loaded in memory that are out of place. Analyzing memory is a common 
activity for me while maintaining an acceptable memory footprint or looking into out-of-memory crashes. I start with 
textures because they are memory intensive and easy to spot and identify issues.

To demonstrate the process, I created a new project in Unity 2021.3.9f1 because this is the version I found the bug in, 
and I know somebody fixed it somewhere between this version and the latest LTS patch. The project has two scenes. The 
first scene contains a canvas with an image component and two IMGUI buttons: one to reproduce conditions for the bug and 
one to load into the second scene. The second scene is empty, and this is where I take a memory snapshot.

Here is the first scene, called `SampleScene` running in play mode. Notice that `Image` is parented to `CanvasA`.
![](sample-scene-play-mode-1.png)

When the "Swap Canvas Parent" button is pressed, `Image` is re-parented to `CanvasB`. That's all it takes to 
trigger the condition for the bug to occur.
![](sample-scene-play-mode-2.png)

Finally, the "Load Empty Scene" button executes a plain old scene transition (non-additive) to the `Empty` scene.
![](empty-scene-play-mode-1.png)

I will take a memory snapshot in this scene, but first, we should talk about editor tools. I use a third-party package 
called [Heap Explorer](https://github.com/pschraut/UnityHeapExplorer#4.1.1) because it provides a "Paths to Root" 
feature that lets you see what static root has a path to a given image or other C# object instance. Unity also has a 
similar package called [Memory 
Profiler](https://docs.unity3d.com/Packages/com.unity.memoryprofiler@1.0/manual/index.html). As far as I know, Memory 
Profiler does not have an equivalent "Paths to Root" feature, and it also is experimental in Unity 2021.3. The 1.0.0 
release is available in Unity 2022. It is a tool to keep an eye on in the future, and it might be appropriate for other 
use cases that Heap Explorer is not.

After you have installed Heap Explorer, you can open it from `Windows/Analysis/Heap Explorer`. Typically, I take memory 
snapshots from a connected player running the game, but for this demonstration it is acceptable to capture an editor 
snapshot. An editor snapshot will include everything the editor has loaded in addition to what play mode has loaded. It 
will produce a lot of noise in the snapshot and make it more difficult to identify real issues. Profile on a connected 
player!

The initial window looks like this.
![](snapshot-1.png)

Click the "Capture" drop-down and select "Capture and Analyze 'Editor'".
![](snapshot-2.png)

After capturing the snapshot, you see a top-level summary.
![](snapshot-3.png)

Click "Investigate" in the top left "Top 20 Native Memory Usage" panel.
![](snapshot-4.png)

Find the image from `SampleScene`. It is called `Star`. Recall that scene containing the image was unloaded, and this 
texture should have been unloaded. We should not be able to find it, but there it is. You can already see "1 Path(s) to 
Root" in the bottom-right, which ends at `GraphicsRegistry`.
![](snapshot-5.png)

You can see the full path when you expand the tree node.
![](snapshot-6.png)

The information presented here is enough to locate all the code keeping the star image loaded. Please note that 
`TestImage` is a class I wrote to extend `Graphic` so I could add some logging. From this point, you can start looking 
at the code. You could start in `GraphicRegistry` to see the methods that manage the dictionary seen in the previous 
image, or you could look at `Graphic` and see where it makes calls into `GraphicsRegistry`. Inside `Graphic`, two 
methods register and unregister themselves with `GraphicsRegistry`. The methods are `OnCanvasHierarchyChanged` and 
`OnTransformParentChanged`, seen below.

~~~
        protected override void OnCanvasHierarchyChanged()
        {
            // Use m_Cavas so we dont auto call CacheCanvas
            Canvas currentCanvas = m_Canvas;

            // Clear the cached canvas. Will be fetched below if active.
            m_Canvas = null;

            if (!IsActive())
            {
                GraphicRegistry.UnregisterGraphicForCanvas(currentCanvas, this);
                return;
            }

            CacheCanvas();

            if (currentCanvas != m_Canvas)
            {
                GraphicRegistry.UnregisterGraphicForCanvas(currentCanvas, this);

                // Only register if we are active and enabled as OnCanvasHierarchyChanged can get called
                // during object destruction and we dont want to register ourself and then become null.
                if (IsActive())
                    GraphicRegistry.RegisterGraphicForCanvas(canvas, this);
            }
        }
~~~
~~~
        protected override void OnTransformParentChanged()
        {
            base.OnTransformParentChanged();

            m_Canvas = null;

            if (!IsActive())
                return;

            CacheCanvas();
            GraphicRegistry.RegisterGraphicForCanvas(canvas, this);
            SetAllDirty();
        }
~~~

When I looked at these two methods, I quickly became suspicious of `OnTransformParentChanged`. It never calls 
`UnregisterGraphicForCanvas`. Could that be the bug? Think back to the `SampleScene` and you will remember the button 
that changes the image's parent from `CanvasA` to `CanvasB`. That is, of course, to test my suspicion of this code. 
Sure enough, after re-parenting the image, the image is unregistered from its old canvas before being registered to 
its new canvas.

I looked at this class in a version of the editor where somebody fixed the bug. For this bug, they didn't make a 
change to `OnTransformParentChanged`, but they made a small change to `OnBeforeTransformParentChanged` to unregister the 
image.

Before:
~~~
        protected override void OnBeforeTransformParentChanged()
        {
            GraphicRegistry.DisableGraphicForCanvas(canvas, this);
            LayoutRebuilder.MarkLayoutForRebuild(rectTransform);
        }

~~~

After:
~~~
        protected override void OnBeforeTransformParentChanged()
        {
            GraphicRegistry.UnregisterGraphicForCanvas(canvas, this);
            LayoutRebuilder.MarkLayoutForRebuild(rectTransform);
        }
~~~

Hopefully, you agree this was a simple bug with a straightforward fix. With the right tools, you can quickly find the 
bug, jump into code, form a hypothesis, test the idea, and identify the root cause for this bug. It's certainly not the 
most challenging bug to investigate and fix, but it does offer a simple introduction to analyzing memory with Heap 
Explorer.

After you have installed Heap Explorer, you can open it from `Windows/Analysis/Heap Explorer`. Typically memory 
snapshots should be taken from a connected player running the game but for the purpose of this demonstration it is fine 
to take an editor snapshot. An editor snapshot is going to include every thing the editor has loaded in addition to what 
play mode has loaded. It's going to produce a lot of noise in the snapshot and makes it more difficult to identify real 
issues. Profile on a connected player!

The initial window looks like this.
![](snapshot-1.png)

Click the "Capture" drop down and select "Capture and Analyze 'Editor'".
![](snapshot-2.png)

You see a top-level summary after the snapshot is captured.
![](snapshot-3.png)

Click "Investigate" in the top left "Top 20 Native Memory Usage" panel.
![](snapshot-4.png)

Find the image from `SampleScene`. It is called `Star`. Recall that scene containing the image was unloaded and this 
texture should have been unloaded. We should not be able to find it but there it is. You can already see in the bottom 
right that there is "1 Path(s) to Root" and it ends at something called `GraphicsRegistry`.
![](snapshot-5.png)

You can see the full path when you expand the tree node.
![](snapshot-6.png)

The information presented here is enough to locate all the code involved in keeping the star image loaded. Please note 
that `TestImage` is a class I wrote to extend `Graphic` so I could add some logging. From this point you can start 
looking at code. You could start in `GraphicRegistry` to see the methods that manage the dictionary seen in the previous 
image or you could look at `Graphic` and see where it makes calls into `GraphicsRegistry`. Inside `Graphic` there are 
two methods that register and unregister themselves with `GraphicsRegistry`. The methods are `OnCanvasHierarchyChanged` 
and `OnTransformParentChanged`, seen below.

~~~
        protected override void OnCanvasHierarchyChanged()
        {
            // Use m_Cavas so we dont auto call CacheCanvas
            Canvas currentCanvas = m_Canvas;

            // Clear the cached canvas. Will be fetched below if active.
            m_Canvas = null;

            if (!IsActive())
            {
                GraphicRegistry.UnregisterGraphicForCanvas(currentCanvas, this);
                return;
            }

            CacheCanvas();

            if (currentCanvas != m_Canvas)
            {
                GraphicRegistry.UnregisterGraphicForCanvas(currentCanvas, this);

                // Only register if we are active and enabled as OnCanvasHierarchyChanged can get called
                // during object destruction and we dont want to register ourself and then become null.
                if (IsActive())
                    GraphicRegistry.RegisterGraphicForCanvas(canvas, this);
            }
        }
~~~
~~~
        protected override void OnTransformParentChanged()
        {
            base.OnTransformParentChanged();

            m_Canvas = null;

            if (!IsActive())
                return;

            CacheCanvas();
            GraphicRegistry.RegisterGraphicForCanvas(canvas, this);
            SetAllDirty();
        }
~~~

When I looked at these two methods, I quickly became suspicious of `OnTransformParentChanged`. It never calls 
`UnregisterGraphicForCanvas`. Could that be the bug? Think back to the `SampleScene` and you will remember the button 
that changes the parent of the image from `CanvasA` to `CanvasB`. That is of course to test my suspicion of this code. 
Sure enough, after re-parenting the image, the image is not unregistered from its old canvas before being registered to 
its new canvas.

I took a look at this class in a version of the Editor where this had been fixed. For this bug, they didn't make a 
change to `OnTransformParentChanged` but they made a small change to `OnBeforeTransformParentChanged` to unregister the 
image.

Before:
~~~
        protected override void OnBeforeTransformParentChanged()
        {
            GraphicRegistry.DisableGraphicForCanvas(canvas, this);
            LayoutRebuilder.MarkLayoutForRebuild(rectTransform);
        }

~~~

After:
~~~
        protected override void OnBeforeTransformParentChanged()
        {
            GraphicRegistry.UnregisterGraphicForCanvas(canvas, this);
            LayoutRebuilder.MarkLayoutForRebuild(rectTransform);
        }
~~~

Hopefully, you agree this was a simple bug with a straightforward fix. I personally found it interesting that with 
the right tools you can quickly find the bug, jump into code, form a hypothesis, test the hypothesis, and identify root 
cause for this bug. It's certainly not the hardest bug to investigate and fix but it does offer a simple introduction to 
analyzing memory with Heap Explorer.
