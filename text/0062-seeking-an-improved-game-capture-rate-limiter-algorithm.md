# Summary

This RFC proposes to implement an improved algorithm for the Game Capture source's rate limiter.

The improvements sought are defined in more detail below, but in short they are: smoother visuals, without significant performance or visual regressions, and with little or no increase in code complexity.

## Background

The Game Capture source _already_ includes a capture rate limiter. The motivation for its design appears to have been primarily on the basis of performance. According to the commit messages[^1][^2] and code comments[^3] from when this feature was implemented, capturing game frames is not zero-cost, especially at very-high framerates (e.g. hundreds of in-game FPS), so a limiter is desirable to keep performance concerns in check.

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

Winnowing down from "more frames than needed" to "less frames" presents an opportunity to selectively capture frames based on their timing, to result in as even a pacing as practical given the inputs and real-time performance constraints.

Because the existing algorithm has not had alternatives extensively explored, there may be similar algorithms that perform somewhat better overall for smoothness and consistency of the output visuals, while maintaining the same rate limits and performance targets.

### Recap of Existing Design

The current implementation[^4][^5] uses a window (a start and end time), where frames before the window are ignored, frames within the window are captured, and frames past the window trigger a re-sync. The window is effectively one "interval" long -- the "interval" is equal to `1000000000 / [the OBS main render FPS]` e.g. ~16666666ns (at 1x capture rate limit and 60FPS OBS), or `(1000000000 / the OBS main render FPS) / 2` e.g. ~8333333ns (at 2x capture rate limit and 60FPS OBS).

Three outcomes are possible, based on the timing of the next frame that the game presents, which Game Capture source intercepts:
- If the next presented frame is too "fast" (precedes the designated window), it is ignored and nothing further happens. Game Capture source waits until the next game frame to do anything further.
- If the next game frame is presented _within_ the window, the frame is captured, and the window is moved forward by exactly one "interval".
- If the next game frame is "late" (past the window), or this is the first frame encountered, the frame is captured, and the capture window is re-synced so it effectively starts at one "interval" _after_ the timestamp when Game Capture encountered the current frame.

[^4]: https://github.com/obsproject/obs-studio/blob/7843a822e01f3cb30995fa90280f8a5fb205e228/plugins/win-capture/graphics-hook/graphics-hook.h#L171-L190
[^5]: https://github.com/obsproject/obs-studio/blob/7843a822e01f3cb30995fa90280f8a5fb205e228/plugins/win-capture/game-capture.c#L825-L843

### Suggested Scope of Optimizations

Relatively lightweight interventions could be pursued, such as tweaking the internal timekeeping of the capture rate algorithm. For example:
- Tweaking the allow-window to be wider or start slightly earlier, and therefore make it biased slightly toward accepting slightly early frames (while keeping the timekeeping "interval," and therefore the effective rate limit, the same)
- Changing the conditions and manner in which the window advances (to avoid e.g. rubber-banding of the window and hopefully keep pacing more even)
- Allowing a budget of an extra number of captures to be allowed when preceding windows were missed

In making these tweaks, the internal accounting of time between allowed captures could be altered, and the opportunity may exist for an algorithm that captures more even pacing of frames more often, all while keeping the rate limiting purpose intact, and with similar performance characteristics.

## Design Goals

For the purposes of this RFC, the replacement algorithm should:
- Select game frames such that the visuals contained in them, as finally rendered in OBS's outputs, are as evenly paced as practical, in pursuit of smooth visuals.
  - A proposed metric is: evenness (lowest variation) of intervals between the original presentation timestamps of game frames that are ultimately visible in OBS's final output frames.
  - Another proposed metric: Fewest doubled game frames (game frames that are visible across multiple consecutive output frames). As, all else being equal, fewer doubled frames should indicate smoother pacing, better adherence to the target output FPS, and less visual judder.
- Be "similarly fast" as the existing rate limiter algorithm. Should not cause issues with OBS or game performance, or result in renderer or encoding lag. Should not pose a performance issue on systems where the current algorithm already works well.
- Respect any configured rate limits, as may prove necessary for performance, as well as to provide users with necessary _control_ over their system performance.
- Keep the complexity of its implementation low.

# Motivation

Smoother-looking, and/or more _reliably_ smooth visuals, when capturing a game/app in OBS, without hurting performance or maintainability.

# Potential Drawbacks

## Pacing/Smoothness Regressions

Any new capture rate limiting algorithm could have regressions of frame pacing smoothness and/or increased skipped/doubled frames, for certain inputs. As the existing algorithm is already doing pretty well, it is unlikely (perhaps impossible) to have a new algorithm that improves many cases without regressing some other case.

Ideally, the regressions should be as small and as uncommon as possible. But the largest improvements should be noticeably large and common enough to justify making the change.

With a "better algorithm," a large proportion of possible inputs might be expected to see no meaningful difference in output, as the existing algorithm handles them fine, but the general effect should be that stutters and jitters, as well as doubled frames in the output, become at least somewhat rarer and less pronounced.

## Performance/Speed Concerns

Any new algorithm could bring performance concerns, as the Game Capture source, including its rate limiter, is on a critical and sensitive path for the performance of OBS for a large proportion of its users. The usefulness of OBS diminishes if it is not performant, and is enhanced if it is more performant.

As such, any new algorithm should use computationally fast or lightweight techniques, and not introduce performance issues generally. A "sophisticated" calculation is not worthwhile if it reduces the number of users for whom OBS is an appropriate choice for real-time use.

## Complexity

Complex code is hard to reason about, hard to debug, hard to maintain and support, and has more surface area for things to go wrong. The ideal algorithm has source code that is relatively simple to read and maintain, and is not prone to behaving unpredictably due to being overly complex.

# Additional Information

## Other Reasons for Un-smooth Visuals (Things which are Out-of-Scope for this RFC)

Several factors influence visual smoothness of OBS's outputs, but are not under Game Capture source's control. While users seeking smooth visuals may wish to address these concerns, they are out of scope for this RFC.

Out of scope for this RFC:
- Playing games at an FPS that isn't an integer factor or multiple of OBS's output FPS will not be perfectly smooth, once finalized into OBS's Constant Frame Rate video outputs of a different FPS. (Game capture algorithm can be optimized with a goal to try and handle this as well as it can, but even factors/multiples improve the best-case of what OBS can possibly do with its current overall design. Game Capture source cannot change this at the moment, regardless of the capture rate limiting algorithm.)
- Games that stutter or jitter heavily may not produce enough visuals to fill a smooth output framerate in real-time. This may result in doubled frames in OBS's output. Game Capture source is limited to selecting from among frames the game successfully renders/presents, and cannot interpolate novel frames or actively re-time individual frames in its current design. Accounting for large jitter to the fullest extent possible may prove impossible without code that is too complex or raises unacceptable performance/reliability drawbacks for this RFC's stated goals.
- Gameplay with large swings in FPS is more difficult to smoothly convert to any Constant Fame Rate. (The Game Capture rate limiting algorithm can be optimized to handle this as well as possible, but cleaner inputs will still improve what is possible in the best case, regardless.)
- Animation errors in-game, such as the game engine failing to cope with uneven frame pacing and simulating wildly smaller/larger time-steps in quick succession. Such animation errors are baked into the game frames' visuals, in a way that would endure even if you displayed the frames at a steady rate. The presentation times of these frames do not give enough information to pace them smoothly, due to the baked-in visual errors. Without exotic optical recognition algorithms (not proposed in this RFC), OBS will not be aware of such visual errors, only the presentation timing will be (indirectly) available to OBS. As such, animation errors cannot be actively optimized for within the scope of this RFC. If their handling are improved by the enhancements called for by this RFC, it will likely be by coincidence.
- Jittery or low-fidelity mouse input resulting in un-smooth camera pans when aiming. Again, this is baked into the visuals of each frame and is separate from frame presentation timing, so OBS cannot directly detect or address this as currently designed, and it is therefore out of scope for the this RFC.
- Game Capture source's timing is not synced to OBS's main render loop. Syncing the two might allow more reliable results, but is unlikely to be achieved without exceeding this RFC's goals for a low complexity intervention. As such, syncing the Game Capture source to the main OBS render loop is out of scope of this RFC.

## Research and Development Background for this Topic, and Tools to Measure Improvement

Frame pacing has been discussed many times in various places online (there is prior art to refer to). There are existing research, technical documentation, engineering explanations, and theory to refer to. There are some tools available to analyze game framerates/pacing, as well as to analyze a game's pacing via video analysis of recordings/streams (suitable to analyze OBS's output directly).

Methods for analyzing OBS's output videos for doubled frames include trdrop.

Tools such as PresentMon are available to analyze game framerates and presentation times directly, the logs of which could be fed into simulations of OBS's Game Capture rate limiting algorithm and main render loop, in order to prototype and test how well such algorithms might work in practice. OBS itself could be modified to record logs or statistics about the captured and ultimately rendered frames.

While frame pacing can appear to be an opaque topic, there are ways of making it more transparent, as well as measure the efficacy of any given approach. This RFC encourages that any new algorithm should be measured for actual concrete improvements, and to better quantify any drawbacks or regressions.
