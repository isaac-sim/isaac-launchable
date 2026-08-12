# SPDX-FileCopyrightText: Copyright (c) 2025 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

FROM nvcr.io/nvidia/base/ubuntu:22.04_20240212

WORKDIR /app

RUN apt update && apt install -y git jq dnsutils curl

# Install Node.js 22
RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && \
    apt-get install -y nodejs

# Set up npm registries for NVIDIA packages
RUN echo 'registry=https://registry.npmjs.org/' > /root/.npmrc && \
    echo '@nvidia:registry=https://edge.urm.nvidia.com/artifactory/api/npm/omniverse-client-npm-local/' >> /root/.npmrc

# Install create-ov-web-rtc-app
RUN npm install -g @nvidia/create-ov-web-rtc-app@2.1.5 --ignore-scripts

# Generate web-viewer-sample
RUN create-ov-web-rtc-app --name="web-viewer-sample" --sample=local-sample --external --outputDir=/app sample-app --framework=react

WORKDIR /app/web-viewer-sample

# Configure Vite to use /viewer/ as base path for nginx reverse proxy and allow all hosts
RUN sed -i "s/export default defineConfig({/export default defineConfig({\n  base: '\/viewer\/',\n  server: {\n    allowedHosts: true\n  },/" vite.config.ts

# Add configurable stream settings with placeholder defaults (replaced at runtime by entrypoint)
RUN sed -i "s/signalingServer: '127.0.0.1',/signalingServer: __SIGNALING_SERVER__,\n            signalingPort: __SIGNALING_PORT__,\n            mediaServer: '__MEDIA_SERVER__',\n            forceWSS: __FORCE_WSS__,/" src/main.tsx

# Install dependencies
RUN npm install --ignore-scripts

COPY entrypoint.sh /entrypoint.sh
ENTRYPOINT ["bash", "/entrypoint.sh"]
