# Video Downloader GUI

A simple and modern desktop application for downloading videos and audio from supported websites through an easy-to-use graphical interface.

## Features

* Clean and simple graphical interface
* Download videos and audio
* Supports multiple websites
* Select available video/audio quality
* FFmpeg support for merging and format conversion
* Download progress display
* No command-line usage required

## Screenshots

Add screenshots of the application here.

## Requirements

* Python 3.10 or newer
* PySide6
* Requests
* FFmpeg

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/adityaksx/yt-dlp-gui.git
cd yt-dlp-gui
```

### 2. Create a virtual environment

#### Windows

```powershell
python -m venv venv
venv\Scripts\activate
```

#### Linux / macOS

```bash
python3 -m venv venv
source venv/bin/activate
```

### 3. Install Python dependencies

```bash
pip install -r requirements.txt
```

## FFmpeg

FFmpeg is recommended for:

* Merging video and audio streams
* Audio extraction
* Format conversion
* Higher-quality downloads that require separate streams

Make sure FFmpeg is installed and available in your system `PATH`.

### Windows

Using Winget:

```powershell
winget install Gyan.FFmpeg
```

You can also install FFmpeg manually and add its `bin` directory to your system `PATH`.

### Linux

Ubuntu/Debian:

```bash
sudo apt install ffmpeg
```

Fedora:

```bash
sudo dnf install ffmpeg
```

### macOS

Using Homebrew:

```bash
brew install ffmpeg
```

## Usage

### Start the application

```bash
python main.py
```

On Linux/macOS, you may need:

```bash
python3 main.py
```

### Basic workflow

1. Launch the application.
2. Enter the URL of the video.
3. Select the desired download options.
4. Choose the output location if available.
5. Start the download.
6. Wait for the download to complete.

## Project Structure

```text
yt-dlp-gui/
├── main.py
├── requirements.txt
├── README.md
├── LICENSE
└── ...
```

> The project structure may change as the application develops.

## Updating Dependencies

To update the downloader engine:

```bash
pip install -U yt-dlp
```

To update all Python dependencies:

```bash
pip install -U -r requirements.txt
```

## Troubleshooting

### FFmpeg not found

Make sure FFmpeg is installed and accessible from the terminal:

```bash
ffmpeg -version
```

If the command is not recognized, add FFmpeg to your system `PATH`.

### Dependency errors

Make sure your virtual environment is activated and reinstall the dependencies:

```bash
pip install -r requirements.txt
```

### Download fails

Possible causes include:

* Unsupported or unavailable URL
* Network connectivity issues
* Website restrictions
* Outdated downloader components
* Missing FFmpeg for operations that require it

Try updating the dependencies:

```bash
pip install -U yt-dlp
```

## Legal Notice

Only download content that you have permission to download or that is otherwise legally available for downloading.

The developer is not responsible for misuse of this application.

## License

This project is licensed under the terms specified in the [LICENSE](LICENSE) file.

## Contributing

Contributions are welcome.

1. Fork the repository.
2. Create a new branch.
3. Make your changes.
4. Test the application.
5. Submit a pull request.

## Disclaimer

This project is an independent application and is not affiliated with or endorsed by any of the websites or services that it may support.
