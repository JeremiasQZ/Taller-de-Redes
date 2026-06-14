FROM node:18-alpine
WORKDIR /usr/src/app
RUN npm install node-media-server
EXPOSE 1935 8000
CMD ["node", "-e", "const NodeMediaServer = require('node-media-server'); const config = { rtmp: { port: 1935, chunk_size: 6000, gop_cache: true, ping: 30, ping_timeout: 60 }, http: { port: 8000, allow_origin: '*' } }; var nms = new NodeMediaServer(config); nms.run();"]
