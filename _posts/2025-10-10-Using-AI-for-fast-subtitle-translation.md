# Comprehensive Subtitle Translation & Burn-in Workflow

This guide outlines the fully automated process for extracting auto-generated subtitles, using AI to create high-quality, natural-sounding translations, and burning the final, styled subtitles into a video file.

-----

## 1\. Required Tools

  * **Python:** With the `srt` library installed (`pip install srt`).
  * **yt-dlp:** For downloading video and subtitle files.
  * **FFmpeg:** For converting subtitle formats and burning them into the video.

-----

## 2\. The Core Python Script (`subtitle_tool.py`)

This is the final version of our script. It extracts numbered text for the AI and rebuilds the subtitle file from a structured JSON input, automatically handling merged timestamps.

```python
# Save this as subtitle_tool.py
import srt
import json
import sys
from datetime import timedelta

def extract_data(srt_file_path, output_txt_path):
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
    print("Running Pass 1: Filtering cumulative subtitles...")
    filtered_subs = []
    if subs:
        for i in range(len(subs) - 1):
            current_content = subs[i].content.strip().replace('\n', ' ')
            next_content = subs[i+1].content.strip().replace('\n', ' ')
            if not next_content.startswith(current_content) or len(next_content) == len(current_content):
                filtered_subs.append(subs[i])
        filtered_subs.append(subs[-1])
    deduped_subs = []
    seen_content = set()
    for sub in filtered_subs:
        content = sub.content.strip().replace('\n', ' ')
        if content not in seen_content:
            deduped_subs.append(sub)
            seen_content.add(content)
    print(f"Reduced from {len(subs)} to {len(deduped_subs)} candidate subtitle entries.")
    print("Running Pass 2: Trimming overlapping text...")
    metadata = []
    text_for_translation = []
    if not deduped_subs:
        print("No subtitles found after cleaning.")
        return
    first_sub = deduped_subs[0]
    first_content = first_sub.content.strip().replace('\n', ' ')
    if first_content:
        metadata.append({'start': first_sub.start.total_seconds(),'end': first_sub.end.total_seconds()})
        text_for_translation.append(first_content)
    for i in range(1, len(deduped_subs)):
        prev_content = text_for_translation[-1]
        current_sub = deduped_subs[i]
        current_content = current_sub.content.strip().replace('\n', ' ')
        if not current_content: continue
        overlap_len = 0
        for j in range(min(len(prev_content), len(current_content)), 0, -1):
            if prev_content.endswith(current_content[:j]):
                overlap_len = j
                break
        trimmed_content = current_content[overlap_len:].lstrip()
        if trimmed_content:
            metadata.append({'start': current_sub.start.total_seconds(),'end': current_sub.end.total_seconds()})
            text_for_translation.append(trimmed_content)
    metadata_file = 'metadata.json'
    with open(metadata_file, 'w', encoding='utf-8') as f:
        json.dump(metadata, f, indent=2, ensure_ascii=False)
    print(f"Metadata for {len(metadata)} cleaned subtitles saved to {metadata_file}")
    with open(output_txt_path, 'w', encoding='utf-8') as f:
        for i, line in enumerate(text_for_translation):
            f.write(f"{i+1} ||| {line}\n")
    print(f"Cleaned text with line numbers saved to {output_txt_path}")

def rebuild_srt(json_input_path, output_srt_path):
    metadata_file = 'metadata.json'
    try:
        with open(metadata_file, 'r', encoding='utf-8') as f:
            metadata = json.load(f)
    except FileNotFoundError:
        print(f"Error: {metadata_file} not found. Run 'extract' first.")
        sys.exit(1)
    try:
        with open(json_input_path, 'r', encoding='utf-8') as f:
            translated_blocks = json.load(f)
    except FileNotFoundError:
        print(f"Error: Input file '{json_input_path}' not found.")
        sys.exit(1)
    except json.JSONDecodeError:
        print(f"Error: Could not decode JSON from '{json_input_path}'. Please ensure it is a valid JSON file.")
        sys.exit(1)
    new_subs = []
    for i, block in enumerate(translated_blocks):
        try:
            original_indices = [x - 1 for x in block['original_lines']]
            content = block['translated_text'].replace('<i>', '').replace('</i>', '') # Option to strip italics
            if not original_indices:
                continue
            start_index = min(original_indices)
            start_time = timedelta(seconds=metadata[start_index]['start'])
            end_index = max(original_indices)
            end_time = timedelta(seconds=metadata[end_index]['end'])
            sub = srt.Subtitle(index=i + 1, start=start_time, end=end_time, content=content)
            new_subs.append(sub)
        except (KeyError, TypeError) as e:
            print(f"Warning: Skipping invalid block in JSON file. Missing key or wrong format: {e}")
            continue
    print(f"Successfully processed {len(new_subs)} final subtitles from JSON input.")
    final_srt_content = srt.compose(new_subs)
    with open(output_srt_path, 'w', encoding='utf-8') as f:
        f.write(final_srt_content)
    print(f"Successfully rebuilt SRT file and saved to {output_srt_path}")

def main():
    if len(sys.argv) < 4:
        print("Usage:")
        print("  To extract: python subtitle_tool.py extract <input.srt> <output.txt>")
        print("  To rebuild: python subtitle_tool.py rebuild <translated.json> <output.srt>")
        return
    command = sys.argv[1]
    if command == 'extract':
        extract_data(sys.argv[2], sys.argv[3])
    elif command == 'rebuild':
        rebuild_srt(sys.argv[2], sys.argv[3])
    else:
        print(f"Unknown command: '{command}'")

if __name__ == "__main__":
    main()
```

-----

## 3\. The Step-by-Step Workflow

### **Phase 1: Source Preparation**

1.  **Download Video & Subtitles:**

      * **Condition:** Start of the project.
      * **Command:**
        ```bash
        yt-dlp --write-auto-sub --sub-lang en --convert-subs srt -o "video.%(ext)s" "YOUTUBE_URL"
        ```
      * **Result:** `video.mp4` and `video.en.srt` (or `.vtt`).

2.  **Convert VTT to SRT (If Necessary):**

      * **Condition:** `yt-dlp` downloaded a `.vtt` file instead of `.srt`.
      * **Command:**
        ```bash
        ffmpeg -i video.en.vtt video.en.srt
        ```
      * **Result:** A `video.en.srt` file ready for our script.

### **Phase 2: AI-Assisted Translation**

3.  **Extract Text for AI:**

      * **Condition:** You have a clean `.srt` file.
      * **Command:**
        ```bash
        python subtitle_tool.py extract video.en.srt original_text.txt
        ```
      * **Result:** `metadata.json` (for timestamps) and `original_text.txt` (for the AI).

4.  **Add Punctuation & Refine (Optional but Recommended):**

      * **Condition:** You want a grammatically correct English base before translation for higher quality results.
      * **Prompt:**
        > You are a professional transcription editor. The following text is from a numbered, cleaned-up automatic speech recognition script, but it is missing all punctuation and proper capitalization.
        > For each numbered line, add periods, commas, question marks, and capitalization as needed to the text after the `|||` separator.
        > **Crucially, you must preserve the line number and the `|||` separator for every single line without alteration.**
        > Here is the text:
        > `[Paste the content of original_text.txt here]`
      * **Action:** Save the AI's output as a new file, **`punctuated_text.txt`**.

5.  **Perform Final Translation & Merging:**

You are an expert translator specializing in creating natural-sounding subtitles. I will provide you with numbered lines of English text from an ASR transcript.
Your task is to:
Translate the text into simplified Chinese.
Combine consecutive lines where it makes a more complete and natural-sounding sentence. You have the creative freedom to decide what to merge, but make sure the combined line is less than 15 chinese characters, because this is for subtitle and need to fit inside a screen for people to read
Remove unnecessary punctuation like commas or periods from the final Chinese text.
Your final output must be a single JSON array of objects. Do not include any other text or explanation outside of this JSON array.
Each object in the JSON array must represent a final subtitle line and contain two keys:
"original_lines": An array of the original integer line numbers that you merged to create this translated line.
"translated_text": A string containing the final, merged translated Chinese text.
Example: If the input is:
* **Action:** Copy the AI's entire JSON response and save it as **`translated_blocks.json`**.
* 
### **Phase 3: Final Video Production**

6.  **Rebuild the Final Subtitle File:**

      * **Condition:** You have created `translated_blocks.json`.
      * **Command:**
        ```bash
        python subtitle_tool.py rebuild translated_blocks.json video.zh.srt
        ```
      * **Result:** A perfectly timed `video.zh.srt` file with natural, merged sentences.

7.  **Handle Italics and Prepare for Styling:**

      * **Condition:** You want to burn subtitles with custom fonts and correctly render italics. This step is highly recommended.
      * **Command:**
        ```bash
        ffmpeg -i video.zh.srt video.zh.ass
        ```
      * **Result:** A `video.zh.ass` file, which is a more powerful format that preserves italics and works best for styling.

8.  **Burn Subtitles into Video:**

      * **Condition:** You have the final `.ass` file and are ready to create the final video.
      * **Command:**
        ```bash
        ffmpeg -i video.mp4 -vf "ass=video.zh.ass:fontsdir=./:force_style='FontName=Noto Sans SC,FontSize=22,PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,BorderStyle=1,MarginV=25'" final_video_burned.mp4
        ```
      * **Customization:**
          * Place your desired font file (e.g., a `.ttf` or `.otf`) in the same folder.
          * Change `FontName` to your font's name.
          * Adjust `FontSize`, `PrimaryColour` (White: `&H00FFFFFF`, Yellow: `&H0000FFFF`), `OutlineColour` (Black: `&H00000000`), and `MarginV` (distance from bottom) as needed.
      * **Result:** Your final video with high-quality, custom-styled subtitles permanently burned in.
