### push an existing repository from the command line
git remote add origin https://github.com/subhash2830/SD.git
git branch -M main
git push -u origin main

### create a new repository on the command line

echo "# SD" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/subhash2830/SD.git
git push -u origin main


final
Local Repo → branch = main
   │
   ├─ git add .
   ├─ git commit -m "message"
   ├─ git push -u origin main
   └─ Result → Folder hierarchy visible in GitHub
