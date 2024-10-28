python
Copy code
import pyaudio
import numpy as np

python
import os
import hashlib
import json
from cryptography.fernet import Fernet
from pathlib import Path
import shutil
import wave

class SecureAudioFile:
    def __init__(self, original_path: str):
        """Initialize a secure audio file with integrity checking and location tracking."""
        self.file_id = hashlib.sha256(os.urandom(32)).hexdigest()
        self.key = Fernet.generate_key()
        self.cipher_suite = Fernet(self.key)
        self.metadata_path = Path.home() / ".secure_audio" / f"{self.file_id}.meta"
        
    def encrypt_and_store(self, source_path: str, dest_path: str):
        """Encrypt audio file and store with metadata."""
        # Read original audio file
        with wave.open(source_path, 'rb') as wav_file:
            audio_data = wav_file.readframes(wav_file.getnframes())
            params = wav_file.getparams()
        
        # Encrypt audio data
        encrypted_data = self.cipher_suite.encrypt(audio_data)
        
        # Create metadata
        metadata = {
            'file_id': self.file_id,
            'current_location': str(dest_path),
            'original_params': {
                'nchannels': params.nchannels,
                'sampwidth': params.sampwidth,
                'framerate': params.framerate,
                'nframes': params.nframes,
                'comptype': params.comptype,
                'compname': params.compname
            },
            'checksum': hashlib.sha256(encrypted_data).hexdigest()
        }
        
        # Ensure metadata directory exists
        self.metadata_path.parent.mkdir(exist_ok=True)
        
        # Write encrypted audio file
        with open(dest_path, 'wb') as f:
            f.write(encrypted_data)
            
        # Store metadata
        with open(self.metadata_path, 'w') as f:
            json.dump(metadata, f)
            
    def move_file(self, new_location: str):
        """Move the file to a new location while maintaining security."""
        if not self.metadata_path.exists():
            raise FileNotFoundError("Metadata not found. File may be corrupted or removed.")
            
        # Read metadata
        with open(self.metadata_path, 'r') as f:
            metadata = json.load(f)
            
        current_location = Path(metadata['current_location'])
        
        # Verify file exists at current location
        if not current_location.exists():
            raise FileNotFoundError(f"File not found at registered location: {current_location}")
            
        # Verify file integrity
        with open(current_location, 'rb') as f:
            current_data = f.read()
            current_checksum = hashlib.sha256(current_data).hexdigest()
            
        if current_checksum != metadata['checksum']:
            raise ValueError("File integrity check failed")
            
        # Move file to new location
        new_path = Path(new_location)
        shutil.move(str(current_location), str(new_path))
        
        # Update metadata
        metadata['current_location'] = str(new_path)
        with open(self.metadata_path, 'w') as f:
            json.dump(metadata, f)
            
    def play_audio(self, output_path=None):
        """Decrypt and play the audio file."""
        if not self.metadata_path.exists():
            raise FileNotFoundError("Metadata not found. File may be corrupted or removed.")
            
        # Read metadata
        with open(self.metadata_path, 'r') as f:
            metadata = json.load(f)
            
        # Read encrypted file
        with open(metadata['current_location'], 'rb') as f:
            encrypted_data = f.read()
            
        # Verify integrity
        if hashlib.sha256(encrypted_data).hexdigest() != metadata['checksum']:
            raise ValueError("File integrity check failed")
            
        # Decrypt data
        decrypted_data = self.cipher_suite.decrypt(encrypted_data)
        
        # If output path provided, write temporary WAV file for playback
        if output_path:
            with wave.open(output_path, 'wb') as wav_file:
                wav_file.setparams((
                    metadata['original_params']['nchannels'],
                    metadata['original_params']['sampwidth'],
                    metadata['original_params']['framerate'],
                    metadata['original_params']['nframes'],
                    metadata['original_params']['comptype'],
                    metadata['original_params']['compname']
    ))

def play_audio(self, output_path=None):
    """Decrypt and play the audio file."""
    if not self.metadata_path.exists():
        raise FileNotFoundError("Metadata not found. File may be corrupted or removed.")
        
    # Read metadata
    with open(self.metadata_path, 'r') as f:
        metadata = json.load(f)
        
    # Read encrypted file
    with open(metadata['current_location'], 'rb') as f:
        encrypted_data = f.read()
        
    # Verify integrity
    if hashlib.sha256(encrypted_data).hexdigest() != metadata['checksum']:
        raise ValueError("File integrity check failed")
        
    # Decrypt data
    decrypted_data = self.cipher_suite.decrypt(encrypted_data)
    
    # If output path provided, write temporary WAV file for playback
    if output_path:
        with wave.open(output_path, 'wb') as wav_file:
            wav_file.setparams((
                metadata['original_params']['nchannels'],
                metadata['original_params']['sampwidth'],
                metadata['original_params']['framerate'],
                metadata['original_params']['nframes'],
                metadata['original_params']['comptype'],
                metadata['original_params']['compname']
            ))
            wav_file.writeframes(decrypted_data)
    
    # Play audio using PyAudio
    p = pyaudio.PyAudio()
    stream = p.open(format=pyaudio.paInt16,
                    channels=metadata['original_params']['nchannels'],
                    rate=metadata['original_params']['framerate'],
                    output=True)
    
    stream.write(decrypted_data)
    stream.stop_stream()
    stream.close()
    p.terminate()

