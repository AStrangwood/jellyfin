# Direct Streaming in Jellyfin

## Overview

Jellyfin server provides three primary methods for delivering media content to clients:

1. **DirectPlay** - The client directly accesses and plays the original media file without any server-side processing
2. **DirectStream** - The server remuxes the media container without transcoding the audio/video streams
3. **Transcode** - The server transcodes the media to a different format/codec for compatibility

This document explains how the **DirectStream** and **DirectPlay** mechanisms work in the Jellyfin server.

## Play Methods

The play method is determined by the `PlayMethod` enum defined in `MediaBrowser.Model/Session/PlayMethod.cs`:

```csharp
public enum PlayMethod
{
    Transcode = 0,
    DirectStream = 1,
    DirectPlay = 2
}
```

### DirectPlay vs DirectStream

Both DirectPlay and DirectStream avoid transcoding, but differ in how the media is delivered:

- **DirectPlay**: The client directly accesses the media file (typically via network share or URL). The server provides minimal processing.
  
- **DirectStream**: The server reads the file and streams it to the client, potentially remuxing it into a different container format while keeping the same audio/video codecs.

The key distinction is identified in `MediaBrowser.Model/Dlna/StreamInfo.cs`:

```csharp
public bool IsDirectStream => MediaSource?.VideoType is not (VideoType.Dvd or VideoType.BluRay)
    && PlayMethod is PlayMethod.DirectStream or PlayMethod.DirectPlay;
```

## Request Flow

### 1. Client Request

When a client requests a video stream, it sends an HTTP request to endpoints like:

- `GET /Videos/{itemId}/stream`
- `GET /Videos/{itemId}/stream.{container}`

These are handled by the `VideosController` in `Jellyfin.Api/Controllers/VideosController.cs`.

### 2. Stream State Creation

The server creates a `StreamState` object containing all information about the request:

```csharp
var state = await StreamingHelpers.GetStreamingState(
    streamingRequest,
    HttpContext,
    _mediaSourceManager,
    _userManager,
    _libraryManager,
    _serverConfigurationManager,
    _mediaEncoder,
    _encodingHelper,
    _transcodeManager,
    _transcodingJobType,
    cancellationToken).ConfigureAwait(false);
```

This state includes:
- Media source information
- Requested codecs and quality
- User information
- Device capabilities
- Whether static streaming is requested

### 3. Direct Stream Decision Logic

The server determines whether to use direct streaming based on several factors:

#### Static Stream with DirectStreamProvider

For live TV and similar scenarios:

```csharp
if (@static.HasValue && @static.Value && state.DirectStreamProvider is not null)
{
    var liveStreamInfo = _mediaSourceManager.GetLiveStreamInfo(streamingRequest.LiveStreamId);
    var liveStream = new ProgressiveFileStream(liveStreamInfo.GetStream());
    return File(liveStream, MimeTypes.GetMimeType("file.ts"));
}
```

#### Static Remote Stream

For HTTP-based media sources:

```csharp
if (@static.HasValue && @static.Value && state.InputProtocol == MediaProtocol.Http)
{
    var httpClient = _httpClientFactory.CreateClient(NamedClient.Default);
    return await FileStreamResponseHelpers.GetStaticRemoteStreamResult(
        state, httpClient, HttpContext).ConfigureAwait(false);
}
```

#### Static File Stream

For local files (DirectPlay/DirectStream):

```csharp
if (@static.HasValue && @static.Value && 
    !(state.MediaSource.VideoType == VideoType.BluRay || 
      state.MediaSource.VideoType == VideoType.Dvd))
{
    var contentType = state.GetMimeType("." + state.OutputContainer, false) 
                      ?? state.GetMimeType(state.MediaPath);

    if (state.MediaSource.IsInfiniteStream)
    {
        var liveStream = new ProgressiveFileStream(state.MediaPath, null, _transcodeManager);
        return File(liveStream, contentType);
    }

    return FileStreamResponseHelpers.GetStaticFileResult(
        state.MediaPath,
        contentType);
}
```

### 4. Response Delivery

#### Static File Response

For regular files, the server uses `PhysicalFileResult`:

```csharp
public static ActionResult GetStaticFileResult(string path, string contentType)
{
    return new PhysicalFileResult(path, contentType) { EnableRangeProcessing = true };
}
```

This enables:
- HTTP Range requests (for seeking)
- Efficient file serving by ASP.NET Core
- Direct file access without copying to memory

#### Progressive Stream Response

For infinite/live streams, `ProgressiveFileStream` is used:

```csharp
var liveStream = new ProgressiveFileStream(state.MediaPath, null, _transcodeManager);
return File(liveStream, contentType);
```

The `ProgressiveFileStream` class (in `MediaBrowser.Controller/Streaming/ProgressiveFileStream.cs`) handles:
- Progressive file reading
- Stream position tracking
- Integration with transcoding jobs

## StreamBuilder and Format Selection

The `StreamBuilder` class in `MediaBrowser.Model/Dlna/StreamBuilder.cs` determines if media can be direct played/streamed:

### Direct Play Profile Matching

```csharp
private (DirectPlayProfile? Profile, PlayMethod? PlayMethod, TranscodeReason TranscodeReasons) 
    GetAudioDirectPlayProfile(MediaSourceInfo item, MediaStream audioStream, MediaOptions options)
{
    // Check if audio codec and container match device profile
    if (directPlayProfile != null)
    {
        if (canDirectPlay) 
        {
            return (directPlayProfile, PlayMethod.DirectPlay, 0);
        }
        else 
        {
            return (directPlayProfile, PlayMethod.DirectStream, transcodeReasons);
        }
    }
}
```

### Key Factors for Direct Streaming

The server considers:

1. **Container Format**: Does the client support the file container (e.g., MP4, MKV)?
2. **Video Codec**: Does the client support the video codec (e.g., H.264, HEVC)?
3. **Audio Codec**: Does the client support the audio codec (e.g., AAC, AC3)?
4. **Subtitles**: Are subtitles compatible or can they be embedded/external?
5. **Network Conditions**: Can the bitrate be sustained?
6. **Device Profile**: Client-specific capabilities and restrictions

If all codecs are compatible but the container isn't, DirectStream is used (remux only).
If codecs are incompatible, Transcode is required.

## Protocol Types

Direct streaming supports multiple protocols defined in `Jellyfin.Data.Enums/MediaStreamProtocol.cs`:

- `http` - Standard HTTP streaming
- `hls` - HTTP Live Streaming (Apple's adaptive streaming)
- `File` - Direct file access (for DirectPlay scenarios)

## Performance Optimizations

### Range Request Support

For static files, range requests are enabled:

```csharp
{ EnableRangeProcessing = true }
```

This allows:
- Efficient seeking in large files
- Partial content delivery (HTTP 206 responses)
- Bandwidth optimization for clients

### Accept-Ranges Header

The server sets appropriate headers:

```csharp
httpContext.Response.Headers[HeaderNames.AcceptRanges] = "none"; // For transcoding
// or range processing is enabled for direct streams
```

## URL Construction

When DirectStream is selected, the `StreamInfo.ToUrl()` method builds the streaming URL:

```csharp
if (IsDirectStream)
{
    sb.Append("&Static=true");
}
```

The `Static=true` parameter indicates to the server that direct streaming should be used.

## Integration Points

### Media Source Manager

`IMediaSourceManager` provides:
- Media source metadata
- Live stream information
- Direct stream providers for live TV

### Encoding Helper

`EncodingHelper` determines:
- Whether stream copy is possible
- Optimal codec selection
- Encoding parameters if transcoding is needed

### Transcoding Manager

Even in DirectStream scenarios, `ITranscodeManager` tracks:
- Active streaming sessions
- Resource cleanup
- Session statistics

## Security Considerations

1. **Path Validation**: File paths are validated to prevent directory traversal
2. **Access Control**: User permissions are checked before streaming
3. **API Keys**: Tokens are included in URLs for authentication
4. **Protocol Restrictions**: Only allowed protocols are supported

## Summary

Direct streaming in Jellyfin provides an efficient method to deliver media content:

- **No transcoding overhead** when codecs are compatible
- **Efficient bandwidth usage** through range requests
- **Fast startup times** without transcoding delays  
- **Quality preservation** by keeping original encoding
- **Flexible delivery** supporting various protocols and client types

The decision flow is:
1. Check if DirectPlay is possible (client can access file directly)
2. Check if DirectStream is possible (remux only)
3. Fall back to Transcoding if necessary

This architecture balances performance, compatibility, and user experience across diverse client devices and network conditions.
