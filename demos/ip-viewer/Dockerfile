# Use a lightweight Python image
FROM python:3.9-slim

# Set working directory
WORKDIR /app

# Copy app code
COPY . .

# Install Flask
RUN pip install flask

# Expose port
EXPOSE 5000

# Run the app
CMD ["python3", "app.py"]
