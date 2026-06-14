FROM alpine:latest
RUN apk add --no-cache ffmpeg
CMD ["ffmpeg", "-re", "-f", "lavfi", "-i", "testsrc=size=640x480:rate=30", "-f", "lavfi", "-i", "sine=frequency=1000:sample_rate=44100", "-c:v", "libx264", "-c:a", "aac", "-f", "flv", "rtmp://rtmp-server/live/prueba"]
