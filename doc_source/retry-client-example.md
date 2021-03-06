# Using the HTTP/2 Retry Client<a name="retry-client-example"></a>

The following is a sample application that uses the retry client to transcribe audio from either a file or your microphone\. You can use this application to test the client, or you can use it as a starting point for your own applications\.

To run the sample, do the following:
+ Copy the retry client to your workspace\. See [Streaming Retry Client Code](streaming-client.md#streaming-retry-client)\.
+ Copy the retry client interface to your workspace\. Implement the interface, or you can use the sample implementation\. See [Streaming Retry Client Interface Code](streaming-client.md#streaming-client-interface)\.
+ Copy the sample application to your workspace\. Build and run the application\.

```
/**
 * COPYRIGHT:
 * <p>
 * Copyright 2018-2019 Amazon.com, Inc. or its affiliates. All Rights Reserved.
 * <p>
 * Licensed under the Apache License, Version 2.0 (the "License").
 * You may not use this file except in compliance with the License.
 * A copy of the License is located at
 * <p>
 * http://www.apache.org/licenses/LICENSE-2.0
 * <p>
 * or in the "license" file accompanying this file. This file is distributed
 * on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either
 * express or implied. See the License for the specific language governing
 * permissions and limitations under the License.
 */
package com.amazonaws.transcribestreaming.retryclient;

import com.amazonaws.transcribestreaming.TranscribeStreamingDemoApp;
import org.reactivestreams.Publisher;
import org.reactivestreams.Subscriber;
import software.amazon.awssdk.auth.credentials.AwsCredentialsProvider;
import software.amazon.awssdk.auth.credentials.EnvironmentVariableCredentialsProvider;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.transcribestreaming.model.AudioStream;
import software.amazon.awssdk.services.transcribestreaming.model.LanguageCode;
import software.amazon.awssdk.services.transcribestreaming.model.StartStreamTranscriptionRequest;

import javax.sound.sampled.LineUnavailableException;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.InputStream;
import java.net.URISyntaxException;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;

import static com.amazonaws.transcribestreaming.TranscribeStreamingDemoApp.getCredentials;

public class StreamingRetryApp {
    private static final String endpoint = "endpoint";
    private static final Region region = Region.US_EAST_1;
    private static final int sample_rate = 28800;
    private static final String encoding = " ";
    private static final String language = LanguageCode.EN_US.toString();

    public static void main(String args[]) throws URISyntaxException, ExecutionException, InterruptedException, LineUnavailableException, FileNotFoundException {
        /**
         * Create Transcribe streaming retry client using AWS credentials.
         */

        TranscribeStreamingRetryClient client = new TranscribeStreamingRetryClient(EnvironmentVariableCredentialsProvider.create() ,endpoint, region);

        StartStreamTranscriptionRequest request = StartStreamTranscriptionRequest.builder()
                .languageCode(language)
                .mediaEncoding(encoding)
                .mediaSampleRateHertz(sample_rate)
                .build();
        /**
         * Start real-time speech recognition. The Transcribe streaming java client uses the Reactive-streams 
         * interface. For reference on Reactive-streams: 
         *     https://github.com/reactive-streams/reactive-streams-jvm
         */
        CompletableFuture<Void> result = client.startStreamTranscription(
                /**
                 * Request parameters. Refer to API documentation for details.
                 */
                request,
                /**
                 * Provide an input audio stream.
                 * For input from a microphone, use getStreamFromMic().
                 * For input from a file, use getStreamFromFile().
                 */
                new AudioStreamPublisher(
                        new FileInputStream(new File("FileName"))),
                /**
                 * Object that defines the behavior on how to handle the stream
                 */
                new StreamTranscriptionBehaviorImpl());

        /**
         * Synchronous wait for stream to close, and close client connection
         */
        result.get();
        client.close();
    }

    private static class AudioStreamPublisher implements Publisher<AudioStream> {
        private final InputStream inputStream;

        private AudioStreamPublisher(InputStream inputStream) {
            this.inputStream = inputStream;
        }

        @Override
        public void subscribe(Subscriber<? super AudioStream> s) {
            if (s.currentSubscription == null) {
                this.currentSubscription = new TranscribeStreamingDemoApp.SubscriptionImpl(s, inputStream);
            } else {
                this.currentSubscription.cancel();
                this.currentSubscription = new TranscribeStreamingDemoApp.SubscriptionImpl(s, inputStream);
            }
            s.onSubscribe(currentSubscription);
        }
    }
}
```