I'm working on the app described in the README.md it's a VUE app written in typscript

I want the child to read parts of the text.

The ocr-ed text is being split into 3 parts.

    I want a dedicated vue component for the textparts called "TextPart". The TextPart should have a special mode / flag "readback" to signal that this part is a part that the child is supposed to read. That means:

    1. Initially don't display the text and do not play the audio
2. instead show the "start recording" button
3. While the recording is running stop the recording when the stop button is pressed or after 10s
4. after stopping send the result for transcription and display the result
5. after receiving the result continue to play the part itself and the next part

