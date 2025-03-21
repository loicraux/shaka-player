# Shaka Player Gap Jumping


## Overview

Gap jumping is a feature to automatically jump over gaps (i.e. missing content)
in a stream.  There are several cases where content may not be present.  This
missing content can cause gaps in the video which will cause the player to stop,
waiting for content that may never exist.  Gap jumping is one of the most
requested features by developers.

There are two kinds of gaps we need to consider: gaps in the manifest and gaps
in the media itself.  Gaps in the manifest can be fixed by simply adjusting the
start and end times of the segments.  We already adjust the segment index to
make the segments continuous.  Gaps in the media cannot be detected early (since
the gaps may not appear in the manifest); they can only be detected once the
media has been appended to the MediaSource and the browser gives us a gap in the
buffered ranges.

## Background

There have been a number of bugs filed asking for this feature.

#### [#180](https://github.com/shaka-project/shaka-player/issues/180)
One of the first bugs asking for gap jumping.  The author talks about how live
streams can commonly lose source content (e.g. from network outages or weather).
This can cause the packagers to introduce gaps in either the media segments or
the manifest.

#### [#555](https://github.com/shaka-project/shaka-player/issues/555)
This one is also asking about live streams and outages.  The author discusses
several options for dealing with interruptions in the live streams.  The
suggestions were:

- Use 404s to indicate a segment does not exist on the server.
- Use “short Periods” (a Period with no content, but @duration) to indicate a gap.
- Fill with dummy content.

It is brought up that using 404 for missing segments can cause problems since we
can’t tell if a segment will never be available or is just not created yet.
Also that dummy content is usually generated by the server; however, the author
notes that this may not be feasible.

#### [#661](https://github.com/shaka-project/shaka-player/issues/661)
This content with small gaps in the media that result in playback stalling.
Chrome and Firefox will actually play through the gap, but we don’t allow it
since it wouldn’t work on Safari.


## Design

If there are gaps in the media (e.g. missing segments), they should not appear
in the manifest.  If there are missing Periods, then the @start attribute should
appear in the later Period to indicate a gap.  If there are gaps in the
manifest, they will be logged and removed by the manifest parser.  Internally,
the segment index will be continuous.  Even if there is a whole Period missing
that is minutes long, the segment index will be continuous.  This means that the
previous Period will be extended to cover the gap and the last segment of that
Period will also be extended.

Keeping the segment index continuous allows most gap jumping logic to be in
Playhead.  StreamingEngine will not know about gaps and will simply buffer
segments like before.  It will buffer segments independently of whether there
are gaps in the media.  Then if Playhead detects a gap in the media, it will
jump it there.

### Manifest Parser

The manifest parser should remove all gaps from the index.  The segment index
should be continuous, even for large gaps.  We are not going to support “empty
periods”; all Period elements should have content.  However, there can be gaps
between segments or between Period elements.  If the gap is larger than a
threshold, a warning is printed.

### Streaming Engine

StreamingEngine will not require any changes.  Since the segment index is
continuous, it will not see any gaps and just buffer like normal.  We will need
to modify the buffering logic a little.  We will calculate the amount buffered
based on the content buffered (i.e. we will ignore the time in the gap).  This
ensures smooth playback with gap jumping.

Like now, it will continue buffering a sequence of segments from the index.  It
will only buffer segments in media order, waiting for any segments that appear
in the index but are not ready on the server (404).  This ensures that any gap
in the buffered ranges is caused by gaps in the content, not by content that
will be available later.

We already fetch the segment before the target time when seeking.  We also only
adjust the end time of a segment, so the start time is “close” to the real time.
This means that when we seek, there will always be content buffered before the
playhead.  Once we have buffered past the playhead, we will know if we have
seeked into a gap.

### Playhead

The Playhead will hold the logic for gap jumping.  When the video plays into a
gap, the browser will do two things: (1) fire a `waiting` event and (2) change
the readyState to 2 (`HAVE_CURRENT_FRAME`) or 1 (`HAVE_METADATA`) if this was a
seek.  To handle this, the Playhead will listen to the `waiting` event and
update the current time to just past the gap.

When we seek, StreamingEngine will clear the buffer.  This will cause a
`waiting` event to fire,  which we ignore since nothing will be buffered.  Once
StreamingEngine starts buffering new content, we need to determine if we are
still in a gap.  If we are inside a gap, the ready state will be 1 or 2 (it will
be 3 or 4 while playing).  We will poll the ready state and if it is 1 or 2, we
are inside (or near) a gap and will need to jump it.  If the ready state is 3 or
4, we can play and we don’t need to do anything more now.

Playhead will only change the current time if we are in a gap, or close to it.
If the first buffered range is slightly ahead of the playhead, we will jump it;
it will also jump if there are multiple buffered ranges and the playhead is
close to a gap.

Playhead will also handle firing the event for large gaps.  Event firing is
synchronous, so Playhead will fire the event and then jump the gap if needed.
