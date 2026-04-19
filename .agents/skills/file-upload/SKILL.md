# File Upload Patterns

Skill for handling file uploads with validation, storage, and progress tracking.

## Backend: Multipart Upload Handler

```typescript
import { Request, Response, NextFunction } from 'express';
import multer, { FileFilterCallback } from 'multer';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import path from 'path';
import crypto from 'crypto';

const s3 = new S3Client({ region: process.env.AWS_REGION });

const ALLOWED_MIME_TYPES = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf'];
const MAX_FILE_SIZE = 10 * 1024 * 1024; // 10MB

const fileFilter = (_req: Request, file: Express.Multer.File, cb: FileFilterCallback) => {
  if (ALLOWED_MIME_TYPES.includes(file.mimetype)) {
    cb(null, true);
  } else {
    cb(new Error(`File type ${file.mimetype} not allowed`));
  }
};

export const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: MAX_FILE_SIZE },
  fileFilter,
});

export async function uploadToS3(file: Express.Multer.File, folder = 'uploads'): Promise<string> {
  const ext = path.extname(file.originalname);
  const key = `${folder}/${crypto.randomUUID()}${ext}`;

  await s3.send(new PutObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: key,
    Body: file.buffer,
    ContentType: file.mimetype,
    // Never trust client-provided filenames in metadata
    Metadata: { originalName: Buffer.from(file.originalname).toString('base64') },
  }));

  return `https://${process.env.S3_BUCKET}.s3.amazonaws.com/${key}`;
}

export async function getPresignedUrl(key: string, expiresIn = 3600): Promise<string> {
  return getSignedUrl(s3, new PutObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: key,
  }), { expiresIn });
}

export function handleUpload(folder?: string) {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      if (!req.file) return res.status(400).json({ error: 'No file provided' });
      const url = await uploadToS3(req.file, folder);
      res.json({ url });
    } catch (err) {
      next(err);
    }
  };
}
```

## Frontend: Upload Hook with Progress

```typescript
import { useState, useCallback } from 'react';

interface UploadState {
  progress: number;
  url: string | null;
  error: string | null;
  uploading: boolean;
}

export function useFileUpload(endpoint: string) {
  const [state, setState] = useState<UploadState>({
    progress: 0,
    url: null,
    error: null,
    uploading: false,
  });

  const upload = useCallback(async (file: File) => {
    setState({ progress: 0, url: null, error: null, uploading: true });

    const formData = new FormData();
    formData.append('file', file);

    return new Promise<string>((resolve, reject) => {
      const xhr = new XMLHttpRequest();

      xhr.upload.addEventListener('progress', (e) => {
        if (e.lengthComputable) {
          setState(prev => ({ ...prev, progress: Math.round((e.loaded / e.total) * 100) }));
        }
      });

      xhr.addEventListener('load', () => {
        if (xhr.status >= 200 && xhr.status < 300) {
          const { url } = JSON.parse(xhr.responseText);
          setState({ progress: 100, url, error: null, uploading: false });
          resolve(url);
        } else {
          const error = 'Upload failed';
          setState(prev => ({ ...prev, error, uploading: false }));
          reject(new Error(error));
        }
      });

      xhr.addEventListener('error', () => {
        const error = 'Network error during upload';
        setState(prev => ({ ...prev, error, uploading: false }));
        reject(new Error(error));
      });

      xhr.open('POST', endpoint);
      xhr.send(formData);
    });
  }, [endpoint]);

  const reset = useCallback(() => {
    setState({ progress: 0, url: null, error: null, uploading: false });
  }, []);

  return { ...state, upload, reset };
}
```

## Usage

```typescript
// Route setup
app.post('/api/upload/avatar', upload.single('file'), handleUpload('avatars'));

// React component
function AvatarUpload() {
  const { upload, progress, url, error, uploading } = useFileUpload('/api/upload/avatar');

  return (
    <div>
      <input type="file" accept="image/*" onChange={e => e.target.files?.[0] && upload(e.target.files[0])} />
      {uploading && <progress value={progress} max={100} />}
      {url && <img src={url} alt="Uploaded avatar" />}
      {error && <p className="error">{error}</p>}
    </div>
  );
}
```
