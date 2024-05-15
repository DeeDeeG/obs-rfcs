# Summary

This RFC proposes to research, identify, and implement an improved algorithm for the Game Capture source's rate limiter.

The improvements sought are defined in more detail below, but in short they are: overall smoother visuals and more-consistent frame pacing, without major performance or visual regressions, and with little or no increase in code complexity.

## Background

The Game Capture source already includes a capture rate limiter. The motivation for its design appears to have been primarily on the basis of performance -- According to the commit messages[^1][^2] and code comments[^3] from when this feature was implemented, capturing game frames is not free, performance-wise, especially at very-high framerates (e.g. hundreds of in-game FPS), so a limiter is desirable to keep performance concerns in check.

[^1]: https://github.com/obsproject/obs-studio/commit/56899df08f4af1d768a3381850fc9e94d5253d8e
    > \- An optimization option to limit framerate to the current OBS framerate to improve capture performance (mostly useful for compatibility capture)
[^2]: https://github.com/obsproject/obs-studio/commit/ec5059cee1d7f140e942bf7ad45645b23dc31dc3
    > For game capture, if a game is running at for example 800 FPS and limit capture framerate is off, it would try to capture all 800 of those frames, dramatically reducing performance more than what would ever be necessary.
    >
    > When limit capture framerate is off, instead of capturing all frames, capture frames at an interval of twice the OBS FPS, identical to how OBS1 works by default.  This should greatly increase performance under that circumstance.
[^3]: https://github.com/obsproject/obs-studio/blob/7843a822e01f3cb30995fa90280f8a5fb205e228/plugins/win-capture/game-capture.c#L834-L837
    > Always limit capture framerate to some extent.  If a game
    > running at 900 FPS is being captured without some sort of
	  > limited capture interval, it will dramatically reduce
	  > performance. */

## Opportunity

Winnowing down from "more frames than needed" to "less frames" presents an opportunity to selectively capture frames based on their timing, to result in as even a pacing as practical (given the inputs, and considering the real-time performance constraints involved).

Because the existing algorithm has not had alternatives to it extensively explored, there may be similar algorithms that could perform somewhat better overall for smoothness and consistency of the output visuals, while maintaining roughly the same rate limits and performance.

### Recap of Existing Design

The current implementation[^4][^5] uses a window (a start and end time), where frames before the window are ignored, frames within or past the window are captured, and any frames past the window trigger a re-sync.

#### Capture Interval / "Window" Length

The window size is effectively one "interval" long in nanoseconds -- the "interval" in OBS is currently equal to either `1000000000 / [OBS render FPS]` (e.g. ~16666666 ns at 1x capture rate limit and 60FPS OBS render loop), or `(1000000000 / [OBS render FPS]) / 2` (e.g. ~8333333 ns at 2x capture rate limit and 60FPS OBS render loop).

[^4]: https://github.com/obsproject/obs-studio/blob/7843a822e01f3cb30995fa90280f8a5fb205e228/plugins/win-capture/graphics-hook/graphics-hook.h#L171-L190
[^5]: https://github.com/obsproject/obs-studio/blob/7843a822e01f3cb30995fa90280f8a5fb205e228/plugins/win-capture/game-capture.c#L825-L843

#### How the Window Advances

For each new app/game frame "presented" through the graphics APIs (D3D, OpenGL, Vulkan...), three outcomes are possible:
- If the presented frame is too "fast" (precedes the current capture window), the game/app frame is ignored and nothing further happens. Game Capture source waits until the next game/app frame to do anything further.
- If the presented frame is presented _within_ the window, the frame is captured, and the window is moved forward by exactly one "interval".
- If the presented frame is "late" (past the window), or this is the first frame encountered, the frame is captured, and the capture window is re-synced so it effectively starts at one interval width _past_ the timestamp when Game Capture encountered the current frame. (Such that the next presented frame must be _at least_ one interval width older than this current frame, or it will be ignored.)

### Suggested Optimizations

_(NOTE: This section does not contain requirements for the implementation, but is given as an illustration or suggestion of areas that could be optimized. Other techniques may be worthwhile if they meet the design goals and motivation of this RFC.)_

Relatively lightweight interventions could be pursued, such as tweaking the internal timekeeping of the capture rate algorithm. For example:
- Tweaking the allow-window to be wider or start slightly earlier, and therefore biasing the limiter slightly toward accepting slightly early frames (while keeping the timekeeping "interval" width the same, and therefore keeping the effective FPS of the rate limit more or less the same, depending on how many additional "late frames"/re-syncs this might cause per second in some scenarios.)
- Changing the conditions and manner in which the window advances (e.g. only advancing by multiples of the window width, not re-syncing to frame time-stamps for late frames, attempting to avoid rubber-banding of the window right after a "late frame" where the capture window re-syncs in the current implementation, to hopefully keep pacing more even).
- Allowing a limited budget of an extra number of captures to be allowed, e.g. when preceding windows were missed (attempting to cut down on the prevalence of doubled frames in OBS's output).

_Other tweaks are likely possible, and need not have been laid out in this section to be valid for the purposes of this RFC. Implementations only need to keep with the design goals and motivation of this RFC (see below), subject also to the usual rules/processes surrounding an OBS RFC. This section is only a set of suggestions, to illustrate the types of changes that one might consider pursuing when implementing this RFC._

## Design Goals

For the purposes of this RFC, the replacement algorithm should:
- Capture game/app frames such that the visuals contained in them, as finally rendered in OBS's outputs, are as evenly paced as practical, in pursuit of smooth visuals.
  - A proposed metric is: given the original presentation timestamps of frames for the app/game being captured, evenness (lowest variation or deviation) of intervals between timestamps from frame to frame, for those frames that are ultimately visible in OBS's final output (e.g. stream or recording). (More consistent "time gap" or "frame time" is better.)
  - Another proposed metric: Fewest doubled app/game frames (app/game frames that are visible across multiple consecutive output frames). As, all else being equal, fewer doubled frames should indicate smoother pacing, better adherence to the target output FPS, and less visual judder. (Fewer doubled frames is better.)
- Be "similarly fast" as the existing rate limiter algorithm. Should not cause new issues with OBS or game performance, or result in additional renderer or encoding lag. Should not pose a performance issue on systems where the current algorithm already works well. (No additional performance issues is the goal, including on "weak" hardware that can currently run OBS to a reasonable extent.)
- Respect any configured rate limits, as is presumed to be necessary for performance reasons, as well as to provide users with any necessary _control_ over their system performance. (Failing to overall adhere to the configured rate limit is bad.)
- Keep the complexity of its implementation low. (As judged subjectively by OBS's maintainers.)

# Motivation

Smoother-looking, and/or more _reliably_ smooth visuals, when capturing a game/app in OBS, without hurting performance or maintainability.

# Potential Drawbacks

The following potential drawbacks should be considered, and avoided, where possible, when implementing this RFC.

## Pacing/Smoothness Regressions

Any new game capture rate limiting algorithm could have regressions of frame pacing smoothness, and/or increased skipped/doubled frames, given certain game/app sessions (with certain frame pacing) as input. As the existing algorithm is already doing pretty well, it is unlikely (perhaps impossible) to have a new algorithm that improves some cases without regressing some other case.

Ideally, the regressions should be as small and as uncommon as possible. Whereas the largest improvements should be noticeably large and/or common enough to justify making the change.

With a "better algorithm," a large proportion of possible inputs (game/app sessions) might be expected to see no meaningful difference in output, as the existing algorithm handles them about as well as can be expected, but the general effect should be that stutters, jitters and doubled frames in the output become at least somewhat rarer and less pronounced, and that the outputs are somewhat improved in the majority of cases wherein any difference is seen whatsoever.

## Performance/Speed Concerns

Any new algorithm could bring performance concerns, as the Game Capture source, including its rate limiter, is on a critical and sensitive path for the performance of OBS for a large proportion of its users. The usefulness of the Game Capture source diminishes if it becomes less performant.

As such, any new algorithm should use computationally fast or lightweight techniques, and not introduce performance issues generally. A "sophisticated" approach is not worthwhile if it reduces the number of users for whom OBS's Game Capture source is an appropriate choice for real-time use.

## Complexity

Complex code is harder to reason about, harder to debug, harder to maintain and support, and has more "surface area" for things to go wrong. The ideal algorithm has source code that is relatively simple to read and maintain, and is not prone to behaving unpredictably in-use due to being overly complex.

# Additional Information

## Other Reasons for Un-smooth Visuals (Things which are Out-of-Scope for this RFC)

Several factors influence visual smoothness of OBS's outputs while being largely outside of Game Capture source's control. While users seeking smooth visuals may wish for these concerns to be addressed, solving them 100% is not considered to be readily achievable by this RFC, and therefore 100% solving these issues is considered beyond the scope of this RFC.

The Game Capture source's rate limiter may be able to partly address some of these concerns, to a large extent these cannot be addressed within the design goals and motivation of this RFC.

Out of scope for this RFC:
- Capturing games/apps that run at an FPS other than a fixed integer factor or multiple of OBS's output FPS will not be perfectly smooth, once finalized into OBS's outputs as a Constant Frame Rate at a different FPS. (The Game Capture source's rate limiting algorithm can be optimized with a goal to try and handle this as well as it can, but integer factors or multiples of FPS improve the best-case of what OBS can possibly do with its current overall design. Game Capture source cannot fundamentally or completely address this issue at the moment, within the design goals and motivation of this RFC. Some imperfections in the output arising from non-integer multiple or factor FPS ratios are still expected after this RFC would be implemented.)
- Games/apps with uneven/uncapped framerate (or which stutter or jitter heavily) may not produce consistently timely frames to fill out an optimally smooth output at constant framerate. This may result in doubled frames in OBS's output, and/or somewhat visually un-smooth output pacing. Game Capture source is limited to selecting from among frames the game successfully renders/presents, and cannot interpolate or generate novel frames, nor actively re-time individual frames, in its current design. (The most-recent captured frame is output effectively as if it had occurred the moment any given final output frame is rendered, regardless of how old the game/app frame was at that time.) This can introduce apparent micro-stutter, if the age of the latest game/app frame is not consistent as of the moments when each of the output frames are rendered. Accounting for large game/app jitter to the fullest extent possible may prove impossible without code that is too complex or raises unacceptable performance/reliability drawbacks for this RFC's design goals and motivation.
- Gameplay sessions with large up/down swings in FPS are in general more difficult to consistently convert to any Constant Fame Rate. (The Game Capture rate limiting algorithm can be optimized to try and handle this as well as it possibly can, but cleaner inputs will still improve what is possible in the best case, given OBS's overall current design, regardless.)
- Animation errors in-game, such as the game engine failing to cope with uneven frame pacing and simulating wildly smaller/larger time-steps in quick succession, will not be correctable after-the-fact by OBS in its overall current design. Such animation errors are baked into the game frames' visuals, in a way that is offset relative to their nominal timestamps. The presentation times of these frames do not give enough information to pace them smoothly, due to the baked-in visual errors. Without some visual recognition approach to reverse-engineer the size of the animation error/offset (which is not proposed in, and likely out of scope of, this RFC), OBS will not be aware of such visual errors; Only the presentation timing of the game/app frames will be (approximately) available to OBS. As such, animation errors cannot be actively optimized for within the scope of this RFC. If the way Game Capture source handles these animation errors is improved by the enhancements called for by this RFC, it will likely be by coincidence.
- Jittery or low-resolution mouse input resulting in un-smooth camera pans when aiming. Like the above bullet point, this is baked into the visuals of each frame, and is separate from frame presentation timing, so OBS cannot directly detect or address this as currently designed, and it is therefore out of scope for this RFC.
- Game Capture source's timing is not synced to OBS's main render loop. Syncing the two might allow more reliable results, but is unlikely to be achieved without exceeding this RFC's goals for a low complexity intervention. As such, syncing the Game Capture source to the main OBS render loop is out of scope of this RFC.
- Likewise, the game/app's frame timing is not synced with OBS's Game Capture source or main render loop. In theory, a consistent number of game/app frames being presented per each iteration of the OBS main render loop is the ideal scenario. Any deviation from this poses some challenge for smooth output pacing, which the Game Capture rate limiting algorithm can try to cope with as well as it can, but cleaner inputs once again improve the best-case of what is possible for OBS to achieve with its current overall design.

## Research and Development Background for this Topic, and Tools to Measure Improvement

Frame pacing has been discussed many times in various places online (there is prior art to refer to). There is existing research, technical documentation, engineering explanations, and theory to refer to. There are some tools available to analyze game framerates/pacing, as well as to analyze a game's pacing via video analysis of recordings/streams (suitable to analyze OBS's output directly).

Methods for analyzing OBS's output videos to detect doubled frames include trdrop.

Tools such as PresentMon are available to analyze game framerates and presentation times directly, the logs of which could be fed into simulations of OBS's Game Capture rate limiting algorithm and main render loop, in order to prototype and test how well such algorithms might work in practice. OBS itself could be modified to record logs or statistics about the presented game/app frames, Game Capture source's captured frames, and the ultimately output frames by OBS.

While frame pacing can be an opaque topic to approach, there are ways of making it more transparent, as well as to objectively measure certain aspects of it. This RFC encourages that any new algorithm should be measured for actual concrete, objective changes in outcome (which could lead to the ultimately subjective improvements sought, namely "smoothness", as well as the objectively measurable "consistency" sought), and to quantify any drawbacks or regressions.
