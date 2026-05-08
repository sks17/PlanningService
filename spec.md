BioSyft provides an automated behavioral analysis platform to streamline animal research in
both academia and industry. Our pipeline already classifies and analyzes key behaviors, but
we’d like to provide our users with insights into the more subtle behaviors not caught by our
supervised model.

The goal is to identify distinct behaviors in a set of videos without manual labeling. This is
accomplished by clustering similar frames across multiple videos based on their behavior
profile, and projecting the frames into 2D coordinates for visualization.
This analysis is already done but we need to build out a single or multiple pages where
users can view and explore the results.

Deliverables
Produce a plan (do not build) for this product. We will walk through the plan together in the
interview. You do not need to produce a written plan, although you are encouraged to.
The plan must include a dashboard to view the frames in low dimensional 2D space, as
well as features to help users understand the clusters and their significance. It is up to
you to design what you think would be best for our users.
We leave the plan’s format entirely to you, but you must include:
● Scope (What the project will and won’t accomplish)
● Goals (how you’re measuring success)
● Tech stack
● Approach & design (rough wireframes/descriptions, how you’d structure the UI/UX)
● Timeline

Assumptions & Constraints
The specification is left intentionally underspecified. If something is unclear but you can make
reasonable assumptions (do not trivialize the project), note it as an assumption and proceed.
● You have about 3 weeks to deliver on the project
● This project will be undertaken solely by you (no additional team members)
● Customers are biotech/pharma companies and academic laboratories

Supplied Data
You are provided with the following data, which are accessible to you through a REST API. You
may or may not find all of them relevant for your plan.
For biometrics, keypoints, and videos, you may assume the data shape & format that is easiest
for you to work with. The general idea is that if you need to, you can connect a variety of data
together based on the video and frame number(s).
Feel free to include arbitrary data in biometrics, but you should note it as an assumption.
● Unsupervised data
○ Coordinates, video they’re from, which frame number they represent, and cluster
classification
● Biometrics data - available for each frame in video
○ Speed, posture, etc.
● Keypoint coordinates - available for each frame in video
○ Coordinate location of nose, neck, paws, etc.
● Videos
○ Each video is around 10-20 minutes, recorded at 100 FPS

User Scenarios
This section provides examples of what the users of your product may be like. This is not a
comprehensive description of the possible user base.
Academia

User: A neuroscience PhD student running a behavioral pharmacology study. They've recorded
~40 videos across two conditions (drug vs. vehicle), uploaded videos to our platform, and the
analysis has finished.
Possible goal: Interpret what each cluster represents behaviorally, then compare cluster usage
between conditions to find drug effects.
They are not a developer. They live in Excel. They've never used a custom analysis dashboard
before.
Industry

User: A study director at mid-sized biotech running a drug discovery program. They've just
completed a study testing 4 candidate compounds against vehicle and a positive control
resulting in ~500 videos. Multiple people will look at the results: the study director who ran the
experiment, the program lead who decides which compounds advance, and a biostatistician
who validates the findings.

Possible goal: Identify which compounds produce behavioral signatures distinct from vehicle.
Generate figures and summary tables for an internal go/no-go meeting in 2 weeks. Results will
inform a multi-million-dollar program decision.
They are technical but not a developer. Comfortable with R or Python for stats. Lives in
GraphPad and PowerPoint for figures. Has used commercial behavioral tools before

