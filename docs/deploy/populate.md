###Populating the Scripts

The Script to populate user data in the database is given in /backend/scripts/populate_users.py
to run this just run the script


Script to populate users and their data from a folder structure.

Expected folder structure:
/users_folder/
├── user1/
│   ├── info.json
│   └── data/
│       ├── file1.txt
│       └── file2.pdf
├── user2/
│   ├── info.json
│   └── data/
│       └── document.docx
└── ...

info.json format:
{
  "name": "John Doe",
  "password": "P@ssw0rd123",
  "email": "johndoe@example.com"
}
"""