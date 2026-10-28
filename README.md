# Episode-52: NPU Testing

Rocky discusses his findings with a "Copilot PC" and an NPU

📺 YouTube video: https://youtu.be/yPl0m0er7AMOct

🏠 Code it with AI Home Page: https://codeitwithai.com

## The goal

Run a coding agent ([opencode](https://opencode.ai)) against a local LLM on a Snapdragon "Copilot PC", and find out whether the NPU helps.

| Component | Detail |
|---|---|
| CPU | Snapdragon X2 Elite (X2E78100), 12 Oryon cores, ARM64 |
| GPU | Qualcomm Adreno X2-85 (integrated) |
| NPU | Qualcomm Hexagon |
| RAM | 32 GB, shared by the CPU, GPU and NPU |

All testing was done in October 2026.

## The short answer

On this PC, the NPU is good at **reading prompts**: it processed long prompts about **2.4 times as fast** as the GPU. It did **not** generate text faster. Generation speed depends mostly on memory bandwidth, and the NPU shares the same memory as the CPU and GPU.

That matters for coding agents, because they send a large prompt (6–7K tokens of instructions and tool definitions) before the model writes anything. The NPU tooling is still young, though: the fastest NPU setup also had unreliable tool calling.

## What we tried

| Approach | Runs on | Result |
|---|---|---|
| LM Studio | CPU only | Its only Windows-on-ARM runtime doesn't use the GPU or NPU |
| llama.cpp, Vulkan build | GPU | Unusable: about 3 tokens/s reading prompts, and it stalled on longer runs |
| llama.cpp, CPU build | CPU | Works, but reads prompts slowly |
| **llama.cpp, OpenCL Adreno build** | **GPU** | **Best reliable setup** |
| Microsoft Foundry Local | NPU | Runs, but slow, and tool calling failed |
| **Qualcomm GenieX** | **NPU** | **Fastest prompt reading; tool calling unreliable** |
| llama.cpp Hexagon backend, built from source | NPU | Skipped: needs Windows test-signing (and possibly Secure Boot off) to load self-signed NPU code |

## GPU: choosing a model

All three models ran on the llama.cpp OpenCL build. This benchmark uses `llama-bench` with 7K tokens already in the context, to match a coding agent's prompt.

| Model | Size | Params used per token | Reading the prompt | Generating text |
|---|---|---|---|---|
| Qwen3-8B, Q4_0 | 4.5 GiB | 8B | 72 tokens/s | 12 tokens/s |
| Qwen3.6-35B-A3B, UD-IQ4_XS | 16.5 GiB | ~3B | 74 tokens/s | 12 tokens/s |
| **gpt-oss-20b, MXFP4** | **11.3 GiB** | **~3.6B** | **107 tokens/s** | **25 tokens/s** |

In real opencode sessions:

| Model | First turn | Later turns | RAM free while running |
|---|---|---|---|
| Qwen3-8B | 116 s | — | — |
| Qwen3.6-35B-A3B | 141 s | 37 s | 1.3 GB |
| **gpt-oss-20b** | **59 s** | **10 s** | **8.5 GB** |

gpt-oss-20b won because it ships in MXFP4, a 4-bit format that the Adreno OpenCL backend has specially tuned code for. The right format for the hardware mattered more than the parameter count.

### Lesson: run one request at a time

By default, llama-server handles up to 4 requests at once, and the Adreno OpenCL backend handles that badly. opencode sends two requests at the start of a session (the main prompt and a request for the session title). With the default setting, generation dropped below 1 token/s. With `-np 1` (one request at a time), speed stayed normal.

## NPU attempt 1: Microsoft Foundry Local

[Foundry Local](https://github.com/microsoft/Foundry-Local) (v0.12.1) runs models on the NPU with no system changes, but only models from Microsoft's catalog that are already converted for the chip. Its NPU models are older ones (Qwen2.5, Phi-3/4-mini, DeepSeek-R1 distills); newer models such as Qwen3 and gpt-oss run only on the GPU or CPU.

Results for **qwen2.5-coder-7b** on the NPU:

| Prompt size | Reading the prompt | Generating text |
|---|---|---|
| 51 tokens | ~200 tokens/s | 13 tokens/s |
| 2.5K tokens | ~270 tokens/s | 10 tokens/s |
| 9.3K tokens | ~108 tokens/s | 5 tokens/s |

Verdict: **not usable for a coding agent.**

- It was slower than gpt-oss-20b on the GPU, even though it's a much smaller, weaker model.
- It didn't reuse prompts it had already processed, so every turn would re-read the whole conversation.
- Tool calling failed, even though the catalog lists it as supported.

## NPU attempt 2: Qualcomm GenieX

[GenieX](https://github.com/qualcomm/GenieX) (v0.9.3) is Qualcomm's open-source runtime (formerly NexaSDK). It includes llama.cpp's Hexagon NPU backend, with the NPU code **signed by Microsoft**, so it runs ordinary GGUF models on the NPU without test-signing. That meant the same gpt-oss-20b model could be compared directly on the NPU and the GPU.

Each setup ran the same benchmark script against its OpenAI-compatible server. The numbers are tokens per second:

| Setup | Reading a 2.2K prompt | Reading a 7.6K prompt | Generating, short context | Generating, 7.6K context |
|---|---|---|---|---|
| **GenieX, NPU** | **686** | **632** | 25 | 18 |
| GenieX, GPU | — | 216 | 24 | 19 |
| llama.cpp OpenCL, GPU | 280 | 262 | 28 | 24 |

These are averages over the whole prompt, so they're higher than the `llama-bench` numbers above, which measure only the speed of more tokens on top of an existing 7K context.

In opencode, with the model already loaded:

| Setup | First turn | Follow-up |
|---|---|---|
| GenieX, NPU | 35 s | 7 s |
| llama.cpp OpenCL, GPU | 59 s | 10 s |

The GPU numbers come from an earlier session with a different task, so they're a rough comparison, not a direct one.

Findings:

- **The NPU read prompts about 2.4 times as fast** as the best GPU setup.
- **It generated text about 20% slower** with a long context.
- **It reused prompts it had already processed**, so follow-up turns only sent the new 0.7–1K tokens.
- **Tool calling was unreliable.** In one of two runs, the model printed the raw tool-call text instead of making the call, and a stray "commentary" line appeared before each step. Both suggest GenieX doesn't fully handle gpt-oss's response format yet.
- **"Hybrid" mode (NPU + GPU together) was slower than the NPU alone and crashed** on a 7.6K-token prompt.
- **GenieX's other NPU path** (Qualcomm AI Hub models) has a fixed context, such as 4096 tokens, which is too small for a coding agent.

## Where that leaves us

- **For daily use:** gpt-oss-20b on the llama.cpp OpenCL (GPU) build, because it's reliable.
- **For the future:** GenieX on the NPU. It's already the fastest at reading prompts, and once tool calling is reliable it should make coding agents noticeably more responsive.
- **What an NPU is good for:** processing input quickly and efficiently, such as long prompts and documents. It isn't good for generating text faster, which is limited by memory bandwidth.

# UPDATE: Local LLM Test Videos
These are videos recorded to demonstrate attempts to run different models on a Surface Pro 12 with the Snapdragon X2 Elite, 32 gb RAM, and the onboard GPU or NPU.

In each case, I load the model, then use OpenCode to interact with the model. First with a simple question, then to generate some C# code to print out some ASCII art. The upper-left window shows the running model, the upper-right window shows me using Claude Code to set up the model, and the bottom window shows OpenCode as I interact with the model.

None of them generated the correct ASCII art, but all of them were able to generate C# to generate something.

[Screen Recording 2026-10-23 151702.mp4](https://youtu.be/JGEwxpE1kHY)

This video shows running the qwen3.6 35B-A3B model using the GPU and CPU. It takes a long time to get loaded into memory and parse the initial OpenCode prompt, after which it is slow but perhaps useable.

Screen Recording 2026-10-23 152701.mp4
Using the GenieX runtime from Qualcomm, I run the GPT-OSS-20B model using the NPU. Not fast either, but it didn't spin up the fans on the laptop, so it clearly used a lot less power.

It does output some odd text artifacts while running. Probably due to GenieX, but maybe due to the model. It also did get stuck in the middle, and I had to prompt it to resume.

Compared to qwen3.6, this is faster and more power efficient, but not as fast as GPT-OSS-20B running on the GPU.

Screen Recording 2026-10-23 153304.mp4
Running the GPT-OSS-20B model using the GPU and CPU (not NPU). This was the most successful configuration, probably useful for someone who is fairly patient. It still caused the laptop's fans to spin quite fast, as it uses a lot of power and generates a lot of heat.

It generated ASCII art that looked the best of the three, but still didn't look anything like the characters "AI".

Beyond the videos
I did a whole bunch of other testing as well. Here's a summary of what worked, what didn't, and how fast it was.

Speeds are in tokens per second. "Reading" is how fast the model processes the prompt, which matters a lot for OpenCode because its first prompt is about 7,000 tokens of instructions and tool definitions. "Generating" is how fast it writes the response.

Hosts and runtimes
Host	Hardware	Result
LM Studio	CPU only	Works, but on Windows on ARM its only runtime is CPU-only, so it can't use the GPU or NPU
llama.cpp, OpenCL build	GPU	Best overall. This is the setup I use day to day
llama.cpp, CPU build	CPU	Works, but reads prompts much more slowly
llama.cpp, Vulkan build	GPU	Not usable: about 3 tokens/s reading prompts, and it stalled on longer runs
Qualcomm GenieX	NPU	Fastest at reading prompts and uses much less power, but tool calling is unreliable
Qualcomm GenieX, hybrid mode	NPU + GPU + CPU	Not usable: slower than the NPU alone, and crashed on long prompts
Microsoft Foundry Local	NPU	Runs, but too slow for OpenCode, and tool calling fails
llama.cpp Hexagon backend, self-built	NPU	Not tried: requires turning on Windows test-signing (possibly disabling Secure Boot). GenieX makes it unnecessary
Models on the GPU
All of these used llama.cpp's OpenCL build. Reading and generating speeds were measured with 7,000 tokens already in the context, to match what OpenCode does.

Model	Size	Reading	Generating	OpenCode first turn	OpenCode later turns	Result
Qwen3-8B (Q4_0)	4.5 GiB	72	12	116 s	—	Works, but slow
Qwen3.6-35B-A3B (IQ4_XS)	16.5 GiB	74	12	141 s	37 s	Works, but slow, and left only 1.3 GB of RAM free
gpt-oss-20b (MXFP4)	11.3 GiB	107	25	59 s	10 s	Best overall
gpt-oss-20b wins mostly because of its MXFP4 format, which the Adreno OpenCL backend has specially tuned code for. The model format turned out to matter more than the number of parameters.

GPU vs NPU
These speeds are averages over a whole 7,600-token prompt, so they're higher than the table above (a different benchmark).

Host	Hardware	Model	Reading	Generating	OpenCode first / later turn	Result
llama.cpp OpenCL	GPU	gpt-oss-20b	262	24	59 s / 10 s *	Works reliably
GenieX	NPU	gpt-oss-20b	632	18	35 s / 7 s	Fast and quiet, but tool calls sometimes print as text instead of running
GenieX	GPU	gpt-oss-20b	216	19	not tested	Slower than llama.cpp on the same GPU
GenieX, hybrid	NPU + GPU + CPU	gpt-oss-20b	423 (shorter prompt)	—	—	Crashed on the 7,600-token prompt
Foundry Local	NPU	qwen2.5-coder-7b	~108 (9,300-token prompt)	5	—	Tool calling fails, and no prompt caching
* The llama.cpp OpenCode times are from an earlier session with a different task.

Takeaways
gpt-oss-20b on the GPU through llama.cpp is the best everyday setup. It's reliable with OpenCode's tools, and fast enough if you're patient.
The NPU reads prompts about 2.4 times as fast and doesn't spin up the fans, but it generates about 20% slower once the conversation gets long. GenieX's tool calling with gpt-oss is unreliable, which explains the odd text in the NPU video. Worth retrying with newer GenieX releases.
Foundry Local's NPU models are older and smaller (Qwen2.5, Phi, DeepSeek-R1 distills), and even the 7B coder model was slower than gpt-oss-20b on the GPU. It doesn't cache prompts, so every OpenCode turn would re-read the whole conversation.
Run one request at a time. OpenCode sends two requests at once (your prompt plus one to generate the session title). With llama.cpp's default settings, that dropped generation to under 1 token/s. The -np 1 option fixes it.
Models I considered but didn't test: Gemma 4 26B-A4B (probably similar to Qwen3.6), Gemma 4 E4B (fast but likely too weak for coding), Gemma 4 31B and Qwen3.8 27B (too slow, because they use all their parameters for every token), and GLM-5.3-Flash and Qwen3.8-Flash-Next (too big for 32 GB).
