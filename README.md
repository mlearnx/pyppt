# pyppt: adding figures to Microsoft PowerPoint on-the-fly

pyppt is a python interface to add figures straight from matplotlib to the active slide in Microsoft PowerPoint on-the-fly: without need to save the figure first and without modification of the pptx file on the disk:

![screencast2](https://user-images.githubusercontent.com/1324881/34696423-100b68e2-f4cf-11e7-926a-48d62e970cc1.gif)


## Installation

pyppt could be installed from pypi:

```
pip install pyppt
```

The latest version of pyppt is always available at [GitHub](https://github.com/vfilimonov/pyppt) at the `master` branch.

**Note**, that both python (or IPython notebook) and PowerPoint should be running on the same Windows machine. For support of remote (e.g. hosted in the cloud) notebooks see section [Remote notebooks] below.

## Basic use

### Adding figures

Function `add_figure()` is at the core functionality of the module. For example of the use-case, open some presentation in the PowerPoint and then run in python console / notebook:
```python
import matplotlib.pyplot as plt
import pyppt as ppt
import numpy as np
plt.hist(np.random.beta(2, 5, size=10000), bins=40);
ppt.add_figure('Center')
```
this will make the histogram, save it to the temporary directory, add it to the current slide in the PowerPoint, then erases the temporary file.

**NB!** `add_figure()` will not work (and raise an exception) if the focus in PowerPoint is not on the slide, but in between two slides.

Full definition has the following arguments:
```python
add_figure(bbox=None, slide_no=None, keep_aspect=True, tight=True, delete_placeholders=True, replace=False, **kwargs)
```
`bbox` defines the boundary box for the figure. It could be specified in a number of ways: e.g. via coordinates (`[x, y, width, height]`) of the figure in pixels:
```python
ppt.add_figure([50, 50, 200, 100])
```
or coordinates as a fraction of the slide width / height (if all values of bbox are within the range `[0, 1]`):
```python
ppt.add_figure([0, 0, 0.5, 1])
```
Further there exist a number of presets that are defined by their names (case-insensitive): `'Center', 'Left', 'Right', 'TopLeft', 'TopRight', 'BottomLeft', 'BottomRight'`, that could be also used with the size modifier: e.g. `'CenterL', 'TopRightXXL'`. Dictionary of presets could be modified:
```python
# Complete preset
ppt.presets['corner'] = [0.8, 0, 0.2, 0.2]
ppt.add_figure('Corner')
# Size and modifier
ppt.preset_sizes['S'] = [0.2, 0.3, 0.6, 0.6]
ppt.preset_modifiers['top'] = [0, 0, 1, 0.5]
ppt.add_figure('TopS')
```
Finally, if `add_figure()` is called with empty `bbox` argument:
```python
ppt.add_figure()
```
then it will look for an empty picture placeholder on the current slide and will fill the first one found with the figure. If no such placeholders is found, then `bbox='Center'` will be used.

By default figure is added to the current slide unless the slide number is specified (argument `slide_no`, following VBA conventions, indexing of slides starts at 1). The aspect ratio of the figure is kept by default (argument `keep_aspect=True`), so the `bbox` dimensions are respected. If `keep_aspect` is set to `False`, the figure will be stretched to fill `bbox` completely.

Argument `delete_placeholders` which is set to `True` by default, defines whether empty placeholders will be kept or not. There're three ways of how to deal with them:
* delete them all (`delete_placeholders=True`). In this case everything, which does not have text or figures will be deleted. So if you want to keep some of them - you should add some text there before calling `add_figure()`.
* keep the all (`delete_placeholders=False`). In this case, all of empty placeholders will be preserved even if they are completely hidden by the added figure.
* The only exception is when `bbox` is not provided (`bbox=None`). In this case the figure will be added to the first available empty placeholder (if found) and keep all other placeholders in place even if `delete_placeholders` is set to `True` (note, that in this case, bbox could be not respected).

Such convention is a workaround of the way how Microsoft PowerPoint treat placeholders. When the picture is added to the slide (even using COM methods), if there're empty placeholders, it will be assigned to the first one available. I.e. it will be placed correctly (`bbox` will be respected), but "internally" it will be contained in the placeholder. I.e. the placeholder will disappear and could not be used for something else anymore (however when the figure will be deleted from the slide, it will appear again).

Last argument `replace` extends functionality `add_figure()` to both adding and replacing figures with the same interface. If it is set  to `True`, then before adding picture it will first check if there're any other pictures on the slide that overlap with the target `bbox` (defined either with coordinates or string). If such pictures are found, then the picture on the slide, that overlap the most, will be replaced by the adding picture, taking its position (i.e. method will act like `replace_figure()` and target `bbox` will be ignored). If no such pictures found - method will add figure in the usual way.

This does not substitute `replace_figure()` and is not very versatile in choosing the picture to replace, but it is mostly convenient when one wants to edit figure on the slide with multiple iterations: first adding draft and then working out the style by simple re-evaluation of the cell in IPython notebook, without need to comment `add_figure()` and add `replace_figure()`.

Finally, `**kwargs` are passed to `plt.savefig()`, so the change of dpi could be done via:
```python
ppt.add_figure('Center', dpi=300)
```


### Replacing figures

Function `replace_figure()` does the trick:
```python
replace_figure(pic_no=None, left_no=None, top_no=None, zorder_no=None, slide_no=None, keep_zorder=True, **kwargs)
```
What it does: it tries to identify the figure which should be deleted using arguments `pic_no, left_no, top_no` or `zorder_no`, saves its position, deletes it from the slide and then calls `add_figure(..., **kwargs)`. By default (unless `keep_zorder` is set to `False`), it preserves z-order position of the picture.

The main inconvenience here is how to choose the figure which is to be replaced. There're no smooth and perfect way for doing this without pointing and clicking on the picture. The best case if there's only one picture on the slide, then
```python
ppt.replace_figure()
```
will do the work. Otherwise the picture could be identified in one of four ways:
* According to the position in the internal list of objects of the slide (`pic_no`): all newer pictures will have higher number. However after a couple of deletes, this won't be intuitive any more
* According to the position from the left (`left_no`): Pictures will be ranked according to the x-coordinate of the left side.
* According to the position from the top (`top_no`): Pictures will be ranked according to the y-coordinate of the top side.
* According to the z-order of the picture (`zorder_no`). Top-most picture (the one on the front) will have the number of 1, etc.

Note, that indexing starts at 1 here, but negative indices are treated in the python way, i.e. `-2` means "second last".

For example, this call:
```python
ppt.replace_figure(left_no=2)
```
will replace the second picture from the left, and this one:
```python
ppt.replace_figure(zorder_no=-1)
```
will replace the back-most picture.

Indeed, there're a lot of situations when such simple rules won't really allow to identify the picture by just looking at the slide, especially when there're many pictures on the slide. Well.. Just live with it. Or PR's and suggestions are always welcome!


### Syntactic sugar

Finally, pyppt hijacks matplotlib, so `add_figure()` and `replace_figure()` are accessible from the `matplotlib.pyplot` directly, e.g.:
```python
import matplotlib.pyplot as plt
import pyppt
import pandas as pd
pd.Series(pd.np.random.randn(100)).cumsum().plot()
plt.add_figure('Center')
```


## Extra features

A few methods allow to set title and subtitle:
```python
ppt.set_title(title, slide_no=None)
ppt.set_subtitle(subtitle, slide_no=None)
```
or bring title to front:
```python
ppt.title_to_front(slide_no=None)
```
These functions take slide number as an argument, if it is not provided, current slide will be used.

Active slide could be changed using the method
```python
ppt.goto_slide(slide_no)
```

New slide could be added using the method:
```python
ppt.add_slide(slide_no=None, layout_as=None, make_active=True)
```
where new slide will be added after the slide number `slide_no` and will have layout as the slide number `layout_as`. As elsewhere - `None` indicates the current slide. For example `add_slide(layout_as=1)` will add a new slide after the current using the title slide as a template. Argument `make_active` defines whether the new slide should be brought to focus or not.

Further some metadata could be extracted from the presentation:
* slide dimensions in pixels: `get_slide_dimensions()`;
* notes from all slides: `get_notes()`;
* coordinates of all figures on the slide in the format `[[x, y, width, height], ...]`: `get_image_positions(slide_no=None)`.

Generic version of `get_image_positions()` retrieves coordinates of all objects on the slide (including empty placeholders):
```python
>>> ppt.get_shape_positions(slide_no=None)
[[36, 21, 648, 90, 14],
 [122, 126, 475, 356, 14]]
```
In this case output will have format of `[[x, y, width, height, type], ...]`. Here `type` represents [MsoShapeType](https://msdn.microsoft.com/en-us/library/aa432678.aspx) and the is accessible via
```python
ppt.msoShapeTypeInt[14]
```

## Remote notebooks

pyppt connects to the running instance of Microsoft Powerpoint using COM interface, thus both pyppt and Powerpoint have to be running on the same Windows machine. However it often happens that IPython notebooks are hosted on some remote server (e.g. in cloud) or in a virtual machine; so here the regular connection would not be possible. In this case connection to Microsoft office could be established within the client-server version of pyppt.

Below: **local machine** denotes Windows machine where the Powerpoint is running as well as the browser which accesses IPython notebook. **Remote machine** will denote the actual server where IPython is running (e.g. hosted in the cloud).

Client-server module requires a few extra dependencies. On the local machine install Flask and Flask-CORS:
```
pip install flask flask-cors
```
On the remote machine install requests, unless intend to use javascript version of pyppt-client (see below):
```
pip install requests
```

Two options are available: *direct connection* and *javascript embedding* for communicating between client and server. The option B (javascript embedding) should be working in all the cases.

### Option A: Direct connection

If it happens so, that the local machine is exposed to the internet and has visible external address and the remote machine could establish direct connection to local. Here're the necessary steps then:

First, figure out the external IP of local machine (e.g. via [whatismyip service](http://www.whatsmyip.org/)), let it be `xxx.xxx.xxx.xxx`.

Second, start pyppt server on the local machine. In the terminal run:
```
pyppt_server -H xxx.xxx.xxx.xxx
```
By default port `8877` is used, so make sure that the firewall is configured appropriately.

Finally, in the IPython notebook (which is running on the remote machine) instead of `import pyppt as ppt` use:
``` python
import pyppt.client as ppt
ppt.init_client('xxx.xxx.xxx.xxx')
```

Then all the functionality of the pyppt module will be available, such as `ppt.set_title()` or `ppt.add_figure()` or `plt.add_figure()`. The only one difference is that the response from the functions (e.g. `get_slide_dimensions()` or `get_image_positions()`) will be returned in string format.

Note: if both local and remote machines are operating within the same local network, then the local IP address could be used as well (see the output of `ipconfig` in terminal).

### Option B: Javascript embedding

Usually it is the case that the local machine is hidden behind the firewall and/or NAT, and direct connection from remote server would require changes in network settings (and in corporate environment often is not possible due to security policies). In this case, the client could be talking to the server not directly, but via the browser where the IPython notebooks are displayed.

Here the server on the local machine could be started without arguments (it will then listen to `localhost`):
```
pyppt_server
```

And in the IPython notebook (instead of `import pyppt as ppt` use:
``` python
import pyppt.client as ppt
ppt.init_client(javascript=True)
```
After than, again all the standard functionality will be available via `ppt`.

However the output response from the functions (e.g. `get_image_positions()`) will be only printed below the cell where the function is executed, and it would not be possible to assign the output to the variable. (After the cell is executed on Shift-Enter, the global variable `_results_pyppt_js_` will contain response, but *only* after the cell).

### Notes on the remote notebooks

Note 1: Option B would require IPython notebook to be [trusted](https://jupyter-notebook.readthedocs.io/en/stable/security.html), which should be normally the case: all code that and all output generated during an interactive session is trusted.

Note 2: If there's a problem, clearing outputs from the notebook often should be enough. If does not help, try to clear outputs, save the notebook, reload the page and run `init_client` again.

Note 3: The present client/server design with the HTTP server on localhost and injecting javascript to the notebook might be not perfect (and required [a hack](https://github.com/vfilimonov/pyppt/issues/5) to make it smooth), but [had its reasons](https://github.com/vfilimonov/pyppt/issues/3#issuecomment-357808536). Any ideas of improvement are [very welcomed](https://github.com/vfilimonov/pyppt/issues).

Note 4: Calls to the `pyppt_server` are done asynchronously, so you should not count on the order of execution on the server side (PowerPoint). This could be critical when many slides are created and filled with images in the loop - some heavy images could be processed slower than quick `add_slide()` method. For this reason it is suggested to make a delay between iterations, e.g.:
```python
import time
for n in range(10):
    plt.plot(np.cumsum(np.random.randn(1000)))
    ppt.add_slide()
    ppt.add_figure(dpi=300)
    ppt.set_title('%d little drunkard boys went out to dine..' % (n+1))
    time.sleep(1)
```


## Why pyppt?..

...especially since there already exists [python-pptx](https://python-pptx.readthedocs.io/en/latest/) - a great tool for automation of the slide generation. Monthly performance reports with fifteen tables and forty figures? Easy! Just make a template once and run a script every month to fill it with up-to-date information.

But my needs are slightly different. Usually there’s no "template" in research presentations and each one is unique - with its own story, structure, sections, layouts and annotations. My usual workflow is all around Jupyter notebooks and analysis therein; and at the same time I have PowerPoint open, where the story and details are being drafted: slides being reshuffled between sections and appendix, and charts being changed many times before the final version. Two things that I do most often: (i) take the plot from the notebook and paste it into the slide or (ii) replace the plot on the slide with another one. Both take quite some annoying micro-actions: save to file, open folder, drop the image and perhaps resize or right-click, "Change picture...", select a new one...

So all what I actually need is to take the active plot and stick it to the active slide; and similar tool for replacing the figure.

That’s it. pyppt is not designed to be a Swiss-army-knife, it is all about using python together with the WYSWYG-editing PowerPoint presentation - about `plt.add_figure()` and `plt.replace_figure()`.


## What about...

* **...PowerPoint for OSX?** Well... (Un?)fortunately OSX does not really use COM. Much of communications with the running apps could be done using the native AppleScript, but at the moment I'm not up to figuring out how to port the code to use these interfaces. Furthermore, I have a slight hope that under OSX you have better things to do, rather than messing with the PowerPoint. At least it was the case for me.
* **...OpenOffice?** There exist [python-uno bridge](https://wiki.openoffice.org/wiki/Python) to work with the UNO (OO's component model). So it should be possible to modify the code of pyppt to make it work with OpenOffice via UNO. But I've never used OO, neither do I have time and interest for porting the code.
* **...changing the properties, adding shapes, animations, etc.?** Right, that would be great to have! If I find that my personal use-cases worth extending pyppt, I would certainly add functionality.
* **..some references for the PowerPoint objects?** Sure! Most of information could be found at [MSDN](https://msdn.microsoft.com/en-us/vba/vba-powerpoint) or similar VBA reference [on github](https://github.com/OfficeDev/VBA-content). Python module `win32com` makes it pretty trivial to reuse VBA code from these docs.
* **...PR's, suggestions and bug reports?** Always welcome!

## Changelog

### [v0.1.1]

* `replace_figure()` now preserves z-order of the picture.
* `add_figure()` now has argument `replace` that allows adding figure on the first call and then replacing it on all further calls.

### [v0.2]

* Client-server architecture for cloud support.
* [v0.2.1] `goto_slide()` method and default argument `make_active=True` for `add_slide()`.
* [v0.2.2] Fix for `CoInitialize has not been called` issue.


## License

pyppt library is released under the MIT license.
