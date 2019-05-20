
# SceneStats

[Coming Soon] :clapper: A C++/OpenCV-based scene detection library &amp; program.  SceneStats will combine the scene detection capabilities of PySceneDetect + DVR-Scan with an emphasis on efficiency and speed.  Once complete, PySceneDetect will be able to use SceneStats instead of OpenCV for the video analysis to speed up scene detection and take advantage of multicore and/or GPGPU-enabled systems.

-------------------------------------------------------------------------------------------------

Currently, PySceneDetect and DVR-Scan are approaching the limits of their performance due to lack of proper threading.  Although Python is an excellent programming language, after much benchmarking and experimentation, Python does not seem to be an adequate choice for high performance concurrency (multiprocessing has some pitfalls when working with large amounts of data like videos).  Furthermore, the GPU module for OpenCV is not exposed via the Python API, disallowing a significant performance increase opportunity for some users (esp. DVR-Scan).


 ## Integration with PySceneDetect
 
This project aims to combine the capabilities of both PySceneDetect and DVR-Scan but in a much more high performance manner, by first creating a C++ library **libscenestats** and benchmarking it's performance against the current implementations.  There will be some overlap in responsibilities, mainly for ease of testing, as only the frame stats feature from SceneStats will be used, although it will also support simple scene listings as a standalone program.

Unlike PySceneDetect, however, SceneStats will not perform any video splitting - just a stats file is generated, and if a threshold is provided, a scene list is printed to stdout.  If a more reliable method of fast seeking is also implemented (e.g. using ffmpeg directly instead of the OpenCV wrapper), SceneStats *may* also provide the ability to output images along each scene boundary, if a threshold is provided.

Once the implementation is complete and has been verified, the program **scenestats** will be created to export frame metrics into PySceneDetect's statsfile format (CSV-based).  Additionally, it is being considered to add a binary statsfile to save on space for large videos.

Using a statsfile as an intermediate allows PySceneDetect to operate by relying completely on SceneStats, removing the requirement for the Python OpenCV package entirely.  None of the current PySceneDetect interface needs to change, although internally it needs to call `scenestats` to generate a statsfile on disk, load it back into memory, and perform a final listing of scenes based on threshold, before performing the rest of the jobs requested by the user as normal.

Note, however, that entirely removing the dependency on OpenCV from PySceneDetect may not be possible in the short term due to using it to obtain certain metrics from the video (as well as providing the functionality to save images for each scene), although this functionality can eventually be integrated directly into SceneStats if required.  It is also hoped that the save-images command can be replaced with a sequence of calls to `ffmpeg` directly, which will make removing OpenCV from the Python side a lot easier.

Additionally, having the algorithms implemented in both Python and C++ allows for better testing, since PySceneDetect can be invoked to compare it's output with SceneStats to ensure they align.  Once existing functionality has been implemented, it can be benchmarked against the existing Python implementations (PySceneDetect's detect-content and detect-threshold, and DVR-Scan's detect-motion).


 ## Combining PySceneDetect and DVR-Scan
 
It has been decided that DVR-Scan will be discontinued and replaced with a new `detect-motion` option for PySceneDetect.  This will help simplify development, issue tracking, and software releases, while also retaining all functionality that DVR-Scan currently has (plus adding new features).  Once the features of DVR-Scan been ported to PySceneDetect, an additional page in the reference manual will be added specifically to help users transition from DVR-Scan by providing examples of equivalent commands.

PySceneDetect specializes in *cut detection* ("dense" scene detection), where for any given input video made up of frames 1 to N, the list of scenes PySceneDetect outputs covers all frames (1 to N).  This is diametrically opposed to DVR-Scan, which specializes in *detecting frames of interest* ("sparse" scene detection), where for a given input video, DVR-Scan outputs cover only a *sub-set* of the input frames.  That being said, the basic idea for both is the same - take a set of input frames, and output a list of scenes.

This new concept of sparse/dense needs to be introduced to PySceneDetect in order to combine both programs.  This is because dense and sparse scene detectors cannot be combined together, as they would produce conflicting information (rather than combining dense detectors, which will only add new cuts).  Thus, each detector will also need to keep a type tag now, indicating if it is of the sparse or dense type, and only dense detectors can be combined (as they are allowed now).


## Planned Architecture for SceneStats

The scene detection process will be performed as a pipeline, which can take advantage of multicore systems much better.  One thread will be responsible for video decoding, while one or more worker threads will run the actual detection algorithms.  The number of worker threads and the way individual detectors are pipelined will depend on the algorithm in order to create as many opportunities for parallelism as possible (for example, the threshold detector can process 1 frame per thread, whereas content detector requires pairs of adjacent frames).  This also means that scene detection algorithms will be responsible for the frame pipelining/buffering process, although default implementations will be provided which can be re-used for most common use cases.

GPU-accelerated detectors will also be made available where possible.  At first, only certain algorithms will have GPU-accelerated implementations, and those will only apply to the scene detection process (not video decoding).  This will be improved in the future once the basic pipeline is up and running, at which point other detectors may be optimized.


-------------------------------------------------------------------------------------------------

Licensed under BSD 3-Clause (see the LICENSE file for details).

Copyright (C) 2019 Brandon Castellano. All rights reserved.

