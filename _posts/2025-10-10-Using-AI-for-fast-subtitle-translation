-----

## 1\. Required Tools

First, ensure you have the necessary free, command-line tools installed.

  * **Python:** If you don't already have it, install Python from the official website: [python.org](https://www.python.org/).
  * **yt-dlp:** A powerful tool for downloading videos and subtitles.
      * Installation instructions: [yt-dlp GitHub](https://www.google.com/search?q=https://github.com/yt-dlp/yt-dlp%23installation)
  * **FFmpeg:** The industry standard for video processing, needed to attach the new subtitles.
      * Installation instructions: [ffmpeg.org](https://ffmpeg.org/download.html)

-----

## 2\. The Step-by-Step Workflow

### Step 1: Download Video and Auto-Generated SRT Subtitles

This step grabs the source video and the raw, messy subtitle file from YouTube.

1.  Open your command line or terminal.
2.  Run the following command, replacing `"YOUTUBE_URL"` with the video's URL:bash
    yt-dlp --write-auto-sub --sub-lang en --convert-subs srt -o "video.%(ext)s" "YOUTUBE\_URL"
    ```
    *   `--convert-subs srt`: This forces `yt-dlp` to provide a `.srt` file, which our script is built for. [4, 5]

    ```

You will now have two files: `video.mp4` (or similar) and `video.en.srt`.

### Step 2: Extract, Clean, and Number the Subtitle Text

This is where we use the improved Python script to clean the ASR artifacts (duplicates, overlaps) and prepare the text in a numbered format that protects its structure from the AI.

1.  Save the new Python script from Section 3 below as `subtitle_tool.py` in the same folder as your video files.
2.  Before running the script for the first time, you need to install its required library. In your terminal, run:
    ```bash
    pip install srt
    ```
3.  Now, run the main script command:
    ```bash
    python subtitle_tool.py extract video.en.srt original_text.txt
    ```

This command reads `video.en.srt`, intelligently removes the duplicate and overlapping lines, and saves the clean text to `original_text.txt`. Crucially, each line in the output file will be numbered and formatted like `1 ||| Some text...`, which is the key to making this process reliable.

### Step 3: Punctuate and Translate with Gemini (AI-Proofed Method)

Using your refined prompts, we'll process the numbered text. This method forces the AI to treat each line as a distinct item, preventing unwanted merging or splitting.

1.  **First Pass (Punctuation):** Open `original_text.txt` and copy the entire numbered list. Paste it into Gemini with your specialized cleaning prompt.

    **Recommended Cleaning Prompt:**

    > You are a professional transcription editor. The following text is from a numbered, cleaned-up automatic speech recognition script, but it is missing all punctuation and proper capitalization.
    > For each numbered line, add periods, commas, question marks, and capitalization as needed to the text after the `|||` separator.
    > Crucially, you must preserve the line number and the `|||` separator for every single line without alteration.

    > Here is the text:

    > ```
    > [Paste your numbered text from original_text.txt here]
    > ```

2.  **Save the Cleaned Text:** Copy the corrected and still-numbered text from Gemini and save it as `cleaned_text.txt`.

3.  **Second Pass (Translation):** Open `cleaned_text.txt` and copy the now-punctuated numbered list. Paste it into Gemini with your translation prompt.

    **Recommended Translation Prompt:**

    > You are an expert translator specializing in video subtitles. Translate the following numbered English text to simplified Chinese.
    > For each numbered line, translate only the English text that appears after the `|||` separator.
    > Crucially, you must preserve the original line number and the `|||` separator at the beginning of every line. The final output must have the exact same number of lines as the input.

    > Here is the text:

    > ```
    > [Paste your numbered text from cleaned_text.txt here]
    > ```

### Step 4: Review and Save the Final Translation

1.  Copy the final numbered Chinese translation from Gemini.
2.  Paste it into a new plain text file.
3.  **Review the translation for accuracy and flow.** This human check remains essential for quality.
4.  Save the final, reviewed text as `chinese_text.txt`.

### Step 5: Rebuild the Chinese Subtitle File

The script will now parse the numbered Chinese text and merge it with the saved timestamps.

1.  In your terminal, run this command:
    ```bash
    python subtitle_tool.py rebuild chinese_text.txt video.zh.srt
    ```

This command reads your numbered `chinese_text.txt` and the `metadata.json` file created in Step 2, producing a perfectly timed Chinese subtitle file named `video.zh.srt`.

### Step 6: Final Human Touch (Optional but Recommended)

Before attaching the subtitles, it's a great practice to do a final visual check. Tools like **Subtitle Edit** are perfect for this.

  * **Desktop Power:** The free, open-source application `Subtitle Edit` is highly recommended. You can load your video (`video.mp4`) and the new subtitle file (`video.zh.srt`) to see a visual representation of the audio waveform. This makes it incredibly easy to spot and fix any remaining timing issues or make last-minute text corrections. [6, 7]
  * **Online Convenience:** For quick checks without installing software, you can use the **Subtitle Edit Online** tool at `https://www.nikse.dk/subtitleedit/online`. It offers many of the core features for a final polish.

### Step 7: Attach the Final Subtitles to the Video

You have two main options for the final video: soft subtitles (selectable) or hard subtitles (burned-in).

**Option A: Soft Subtitles (Recommended for Flexibility)**
This method embeds the subtitles as a separate track that viewers can turn on or off.

1.  In your terminal, run this command:
    ```bash
    ffmpeg -i video.mp4 -i video.zh.srt -c copy -c:s mov_text -metadata:s:s:0 language=zho "final_video_softsubs.mp4"
    ```

The file `final_video_softsubs.mp4` is now ready for upload to platforms that support subtitle tracks.

**Option B: Hard Subtitles / Burn-in (For Maximum Compatibility)**
This method permanently writes the subtitles onto the video frames, ensuring they are always visible. Your custom command is perfect for controlling the appearance.

1.  In your terminal, run your refined command:
    ```bash
    ffmpeg -i video.mp4 -vf "subtitles=video.zh.srt:force_style='FontName=Arial,FontSize=22,PrimaryColour=&H00FFFFFF'" final_video_burned.mp4
    ```

The file `final_video_burned.mp4` is now ready and will display the subtitles exactly as styled, on any platform.

-----

## 3\. The Final Python Script (`subtitle_tool.py`)

This is the complete, refined script incorporating your numbering logic. It is designed to be robust against AI formatting errors.

**Save this code as `subtitle_tool.py`:**

```python
import srt
import json
import sys
import re
from datetime import timedelta

def find_best_overlap(s1, s2):
    """Finds the length of the longest suffix of s1 that is also a prefix of s2."""
    s1, s2 = s1.lower(), s2.lower()
    max_overlap = 0
    # Iterate from the smaller of the two string lengths down to 0
    for i in range(min(len(s1), len(s2)), 0, -1):
        if s1.endswith(s2[:i]):
            max_overlap = i
            break
    return max_overlap

def extract_data(srt_file_path, output_txt_path):
    """
    Reads an SRT file, cleans ASR artifacts (cumulative and overlapping),
    and extracts the cleaned text with numbering for AI processing.
    """
    try:
        with open(srt_file_path, 'r', encoding='utf-8') as f:
            content = f.read()
    except Exception as e:
        print(f"Error reading file {srt_file_path}: {e}")
        sys.exit(1)

    print(f"Parsing SRT file: {srt_file_path}")
    try:
        subs = list(srt.parse(content))
    except Exception as e:
        print(f"Error parsing SRT file. It may be malformed. Details: {e}")
        sys.exit(1)

    # Pass 1: Filter for "most complete" subtitles to handle cumulative text.
    print("Running Pass 1: Filtering cumulative subtitles...")
    filtered_subs =
    for i in range(len(subs) - 1):
        current_content = subs[i].content.strip().replace('\\n', ' ')
        next_content = subs[i+1].content.strip().replace('\\n', ' ')
        if not next_content.startswith(current_content) or len(next_content) == len(current_content):
            filtered_subs.append(subs[i])
    if subs:
        filtered_subs.append(subs[-1])

    # Pass 2: Remove pure duplicates that might result from Pass 1.
    deduped_subs =
    seen_content = set()
    for sub in filtered_subs:
        content = sub.content.strip().replace('\\n', ' ')
        if content not in seen_content:
            deduped_subs.append(sub)
            seen_content.add(content)
    
    print(f"Reduced from {len(subs)} to {len(deduped_subs)} candidate subtitle entries.")

    # Pass 3: Process for overlaps and generate final text and metadata.
    print("Running Pass 2: Trimming overlapping text...")
    metadata =
    text_for_translation =

    if not deduped_subs:
        print("No subtitles found after cleaning.")
        return

    # Process the first subtitle.
    first_sub = deduped_subs
    first_content = first_sub.content.strip().replace('\\n', ' ')
    if first_content:
        metadata.append({'start': first_sub.start.total_seconds(), 'end': first_sub.end.total_seconds()})
        text_for_translation.append(first_content)

    # Process the rest of the subtitles.
    for i in range(1, len(deduped_subs)):
        prev_content = text_for_translation[-1]
        current_sub = deduped_subs[i]
        current_content = current_sub.content.strip().replace('\\n', ' ')

        if not current_content:
            continue

        overlap_len = find_best_overlap(prev_content, current_content)
        trimmed_content = current_content[overlap_len:].lstrip()

        if trimmed_content:
            metadata.append({'start': current_sub.start.total_seconds(), 'end': current_sub.end.total_seconds()})
            text_for_translation.append(trimmed_content)

    # Save metadata to a JSON file for the rebuild step.
    metadata_file = 'metadata.json'
    with open(metadata_file, 'w', encoding='utf-8') as f:
        json.dump(metadata, f, indent=2, ensure_ascii=False)
    print(f"Metadata for {len(metadata)} cleaned subtitles saved to {metadata_file}")

    # Save the cleaned text to the specified output file with numbering.
    with open(output_txt_path, 'w', encoding='utf-8') as f:
        for i, line in enumerate(text_for_translation):
            f.write(f"{i+1} ||| {line}\\n")
    print(f"Cleaned text with line numbers saved to {output_txt_path}")

def rebuild_srt(translated_text_path, output_srt_path):
    """
    Rebuilds an SRT file using stored metadata and numbered, translated text.
    """
    metadata_file = 'metadata.json'
    try:
        with open(metadata_file, 'r', encoding='utf-8') as f:
            metadata = json.load(f)
    except FileNotFoundError:
        print(f"Error: {metadata_file} not found. Please run the 'extract' command first.")
        sys.exit(1)
    except Exception as e:
        print(f"Error reading metadata file {metadata_file}: {e}")
        sys.exit(1)

    try:
        with open(translated_text_path, 'r', encoding='utf-8') as f:
            translated_lines = f.readlines()
    except Exception as e:
        print(f"Error reading translated text file {translated_text_path}: {e}")
        sys.exit(1)

    # Parse the numbered lines from the translated file.
    translated_blocks =
    for line in translated_lines:
        if '|||' in line:
            # Split only on the first occurrence of the separator
            parts = line.split('|||', 1)
            if len(parts) == 2:
                translated_blocks.append(parts.[1]strip())

    if len(metadata)!= len(translated_blocks):
        print("\\n--- ERROR ---")
        print("Mismatch between number of metadata entries and translated text blocks.")
        print(f"Metadata entries: {len(metadata)}")
        print(f"Translated blocks: {len(translated_blocks)}")
        print("This can happen if the AI translation removed or altered the '|||' separator.")
        print("Please check your translated text file to ensure each line starts with 'number |||'.")
        print("-------------\\n")
        sys.exit(1)

    print("Metadata and translated text counts match. Rebuilding SRT...")
    new_subs =
    for i, meta_item in enumerate(metadata):
        sub = srt.Subtitle(
            index=i + 1,
            start=timedelta(seconds=meta_item['start']),
            end=timedelta(seconds=meta_item['end']),
            content=translated_blocks[i]
        )
        new_subs.append(sub)

    final_srt_content = srt.compose(new_subs)

    with open(output_srt_path, 'w', encoding='utf-8') as f:
        f.write(final_srt_content)
    print(f"Successfully rebuilt SRT file and saved to {output_srt_path}")

def main():
    """Main function to handle command-line arguments."""
    if len(sys.argv) < 2:
        print("Usage:")
        print("  To extract: python subtitle_tool.py extract <input.srt> <output.txt>")
        print("  To rebuild: python subtitle_tool.py rebuild <translated.txt> <output.srt>")
        return

    command = sys.argv[1]
    
    if command == 'extract':
        if len(sys.argv)!= 4:
            print("Usage: python subtitle_tool.py extract <input.srt> <output.txt>")
            return
        extract_data(sys.argv[2], sys.argv[3])

    elif command == 'rebuild':
        if len(sys.argv)!= 4:
            print("Usage: python subtitle_tool.py rebuild <translated.txt> <output.srt>")
            return
        rebuild_srt(sys.argv[2], sys.argv[3])
    else:
        print(f"Unknown command: '{command}'")

if __name__ == "__main__":
    main()
```

```
```
