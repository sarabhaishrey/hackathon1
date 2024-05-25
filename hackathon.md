mkdir csv-upload-app
cd csv-upload-app
npm init -y
npm install express multer csv-parser mongoose socket.io aws-sdk
// server.js
const express = require('express');
const http = require('http');
const socketIo = require('socket.io');
const mongoose = require('mongoose');
const multer = require('multer');
const csvParser = require('csv-parser');
const fs = require('fs');
const path = require('path');
const AWS = require('aws-sdk');

const app = express();
const server = http.createServer(app);
const io = socketIo(server);

// MongoDB setup
mongoose.connect('mongodb://localhost:27017/csvUploadApp', {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});

const UserSchema = new mongoose.Schema({
  email: String,
  name: String,
  creditScore: Number,
  creditLines: Number,
  maskedPhoneNumber: String,
});

const User = mongoose.model('User', UserSchema);

// AWS S3 setup
const s3 = new AWS.S3({
  accessKeyId: process.env.AWS_ACCESS_KEY,
  secretAccessKey: process.env.AWS_SECRET_KEY,
});

const upload = multer({ dest: 'uploads/' });

app.use(express.static('public'));

// Real-time feedback setup
io.on('connection', (socket) => {
  console.log('New client connected');
  socket.on('disconnect', () => {
    console.log('Client disconnected');
  });
});

// File upload route
app.post('/upload', upload.single('file'), (req, res) => {
  const file = req.file;
  const filePath = path.join(__dirname, 'uploads', file.filename);

  // Upload file to S3
  const params = {
    Bucket: 'your-s3-bucket-name',
    Key: file.originalname,
    Body: fs.createReadStream(filePath),
  };

  s3.upload(params, (err, data) => {
    if (err) {
      console.error('Error uploading to S3:', err);
      return res.status(500).send(err);
    }

    // Process CSV file
    const results = [];
    fs.createReadStream(filePath)
      .pipe(csvParser())
      .on('data', (data) => {
        results.push(data);
        io.emit('uploadProgress', { progress: results.length });
      })
      .on('end', () => {
        User.insertMany(results, (err, docs) => {
          if (err) {
            console.error('Error saving to MongoDB:', err);
            return res.status(500).send(err);
          }
          res.status(200).send('File uploaded and data saved successfully.');
        });
      });
  });
});

// Data display route with pagination
app.get('/users', async (req, res) => {
  const { page = 1, limit = 100 } = req.query;
  const users = await User.find()
    .skip((page - 1) * limit)
    .limit(Number(limit));
  res.json(users);
});

server.listen(3000, () => {
  console.log('Server is running on port 3000');
});
npx create-react-app csv-upload-frontend
cd csv-upload-frontend
npm install axios socket.io-client
// CSVUpload.js
import React, { useState } from 'react';
import axios from 'axios';
import io from 'socket.io-client';

const socket = io('http://localhost:3000');

const CSVUpload = () => {
  const [file, setFile] = useState(null);
  const [uploadProgress, setUploadProgress] = useState(0);

  const handleFileChange = (e) => {
    setFile(e.target.files[0]);
  };

  const handleUpload = async () => {
    const formData = new FormData();
    formData.append('file', file);

    socket.on('uploadProgress', (data) => {
      setUploadProgress(data.progress);
    });

    try {
      await axios.post('http://localhost:3000/upload', formData, {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
      });
      alert('File uploaded successfully');
    } catch (error) {
      console.error('Error uploading file:', error);
      alert('File upload failed');
    }
  };

  return (
    <div>
      <input type="file" onChange={handleFileChange} />
      <button onClick={handleUpload}>Upload</button>
      <div>Upload Progress: {uploadProgress}</div>
    </div>
  );
};

export default CSVUpload;
// DataDisplay.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const DataDisplay = () => {
  const [users, setUsers] = useState([]);
  const [page, setPage] = useState(1);
  const [limit] = useState(100);

  useEffect(() => {
    const fetchData = async () => {
      const response = await axios.get('http://localhost:3000/users', {
        params: { page, limit },
      });
      setUsers(response.data);
    };

    fetchData();
  }, [page, limit]);

  const handlePrevious = () => {
    setPage(page > 1 ? page - 1 : 1);
  };

  const handleNext = () => {
    setPage(page + 1);
  };

  return (
    <div>
      <table>
        <thead>
          <tr>
            <th>Email</th>
            <th>Name</th>
            <th>Credit Score</th>
            <th>Credit Lines</th>
            <th>Masked Phone Number</th>
          </tr>
        </thead>
        <tbody>
          {users.map((user) => (
            <tr key={user._id}>
              <td>{user.email}</td>
              <td>{user.name}</td>
              <td>{user.creditScore}</td>
              <td>{user.creditLines}</td>
              <td>{user.maskedPhoneNumber}</td>
            </tr>
          ))}
        </tbody>
      </table>
      <button onClick={handlePrevious}>Previous</button>
      <button onClick={handleNext}>Next</button>
    </div>
  );
};

export default DataDisplay;
// SubscriptionCalculator.js
import React, { useState } from 'react';
import axios from 'axios';

const SubscriptionCalculator = () => {
  const [creditScore, setCreditScore] = useState(0);
  const [creditLines, setCreditLines] = useState(0);
  const [subscriptionPrice, setSubscriptionPrice] = useState(0);

  const calculateSubscriptionPrice = (basePrice, pricePerCreditLine, pricePerCreditScorePoint) => {
    return basePrice + (pricePerCreditLine * creditLines) + (pricePerCreditScorePoint * creditScore);
  };

  const handleCalculate = () => {
    const basePrice = 10; // Base price
    const pricePerCreditLine = 2; // Price per credit line
    const pricePerCreditScorePoint = 0.1; // Price per credit score point

    const price = calculateSubscriptionPrice(basePrice, pricePerCreditLine, pricePerCreditScorePoint);
   
