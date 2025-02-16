# FastAPI Book Management API

## Overview
This project is a RESTful API built with FastAPI for managing a book collection. It provides comprehensive CRUD (Create, Read, Update, Delete) operations for books with proper error handling, input validation, and documentation.

## Features
* Book management (CRUD operations)
* Input validation using Pydantic models
* Enum-based genre classification
* Complete test coverage
* Auto-generated API documentation
* CORS middleware enabled
* Continuous Integration/Continuous Deployment
* Nginx reverse proxy configuration

## Project Structure
```
fastapi-book-project/
├── api/
│   ├── db/
│   │   ├── __init__.py
│   │   └── schemas.py      # Data models and in-memory database
│   ├── routes/
│   │   ├── __init__.py
│   │   └── books.py        # Book route handlers
│   └── router.py           # API router configuration
├── core/
│   ├── __init__.py
│   └── config.py           # Application settings
├── tests/
│   ├── __init__.py
│   └── test_books.py       # API endpoint tests
├── main.py                 # Application entry point
├── requirements.txt        # Project dependencies
└── README.md
```

## Technologies Used
* Python 3.12
* FastAPI
* Pydantic
* pytest
* uvicorn
* Nginx

## Installation

1. Clone the repository:
```bash
git clone https://github.com/hng12-devbotops/fastapi-book-project.git
cd fastapi-book-project
```

2. Create a virtual environment:
```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

3. Install dependencies:
```bash
pip install -r requirements.txt
```

## Running the Application

1. Start the server:
```bash
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

2. Access the API documentation:
* Swagger UI: http://localhost:8000/docs
* ReDoc: http://localhost:8000/redoc

## API Endpoints

### Books
* `GET /api/v1/books/` - Get all books
* `GET /api/v1/books/{book_id}` - Get a specific book
* `POST /api/v1/books/` - Create a new book
* `PUT /api/v1/books/{book_id}` - Update a book
* `DELETE /api/v1/books/{book_id}` - Delete a book

### Missing API
```code 
@router.get("/{book_id}", response_model=Book, status_code=status.HTTP_200_OK)
async def get_book(book_id: int) -> Book:
    book = db.books.get(book_id)
    if book is None:
        return JSONResponse(
            status_code=status.HTTP_404_NOT_FOUND,
            content={"detail": "Book not found"}
        )
    return JSONResponse(
        status_code=status.HTTP_200_OK,
        content=book.model_dump()
    )
```

### Example Response

For a GET request to `/api/v1/books/{book_id}`:
```json
{
  "id": 1,
  "title": "The Hobbit",
  "author": "J.R.R. Tolkien",
  "publication_year": 1937,
  "genre": "Science Fiction"
}
```

If the book is not found:
```json
{
  "detail": "Book not found"
}
```

## Running Tests
```bash
pytest
```

## Deployment

### AWS Deployment with Nginx

#### Step 1: Launch an AWS EC2 Instance
1. Log in to the AWS Console and navigate to the EC2 Dashboard
2. Click Launch Instance
3. Choose an Ubuntu 22.04 AMI
4. Select an instance type (e.g., t2.micro for free-tier users)
5. Configure security groups to allow ports 22 (SSH), 80 (HTTP), and 8000 (FastAPI)
6. Download and save the .pem key securely
7. Launch the instance and connect via SSH:
   ```bash
   ssh -i your-key.pem ubuntu@your-instance-ip
   ```

#### Step 2: Install Dependencies
```bash
sudo apt update && sudo apt install -y nginx python3 python3-venv python3-pip
sudo systemctl start nginx
sudo systemctl enable nginx
```

Verify that Nginx is installed:
```bash
nginx -v
```

#### Step 3: Deploy the FastAPI Application
1. Clone your repository:
   ```bash
   git clone https://github.com/your-repo.git fastapi-app
   cd fastapi-app
   ```
2. Set up a virtual environment and install dependencies:
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   ```
3. Create a systemd service file for FastAPI:
   ```bash
   sudo nano /etc/systemd/system/fastapi.service
   ```
   Add the following content:
   ```
   [Unit]
   Description=FastAPI application
   After=network.target

   [Service]
   User=ubuntu
   WorkingDirectory=/home/ubuntu/fastapi-app
   ExecStart=/home/ubuntu/fastapi-app/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```
4. Start the FastAPI service:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable fastapi
   sudo systemctl start fastapi
   sudo systemctl status fastapi
   ```

#### Step 4: Configure Nginx as a Reverse Proxy
1. Create a new Nginx configuration file:
   ```bash
   sudo nano /etc/nginx/sites-available/fastapi
   ```
   Add the following configuration:
   ```
   server {
       listen 80;
       server_name your-domain-or-ip;

       location / {
           proxy_pass http://127.0.0.1:8000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }
   }
   ```
2. Enable the configuration:
   ```bash
   sudo ln -s /etc/nginx/sites-available/fastapi /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl restart nginx
   ```
3. Test if Nginx is serving the FastAPI app:
   ```bash
   curl http://localhost
   ```

### CI/CD with GitHub Actions

#### Step 5: Set Up CI/CD with GitHub Actions
1. Create `.github/workflows/ci.yml` for Continuous Integration:
   ```yaml
   name: CI
   on: [push, pull_request]
   jobs:
     test:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3
         - name: Set up Python
           uses: actions/setup-python@v3
           with:
             python-version: '3.10'
         - name: Install dependencies
           run: |
             python -m venv venv
             source venv/bin/activate
             pip install -r requirements.txt
         - name: Run Tests
           run: |
             source venv/bin/activate
             pytest
   ```

2. Create `.github/workflows/cd.yml` for Continuous Deployment:
   ```yaml
   name: CD
   on:
     push:
       branches:
         - main

   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - name: Deploy via SSH
           uses: appleboy/ssh-action@master
           with:
             host: ${{ secrets.SERVER_IP }}
             username: ubuntu
             key: ${{ secrets.SSH_PRIVATE_KEY }}
             script: |
               cd /home/ubuntu/fastapi-app
               git pull origin main
               source venv/bin/activate
               pip install -r requirements.txt
               sudo systemctl restart fastapi
               sudo systemctl restart nginx
   ```

#### Step 6: Adding Secrets to GitHub
Go to your repository settings on GitHub and add the following secrets under `Settings -> Secrets and Variables -> Actions`:
* `SERVER_IP`: Your EC2 instance IP
* `SSH_PRIVATE_KEY`: Your private SSH key (from `.pem` file)

## Error Handling
The API includes proper error handling for:
* Non-existent books
* Invalid book IDs
* Invalid genre types
* Malformed requests

## Contributing
1. Fork the repository
2. Create a feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit changes (`git commit -m 'Add AmazingFeature'`)
4. Push to branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## License
This project is licensed under the MIT License - see the LICENSE file for details.

## Support
For support, please open an issue in the GitHub repository.
