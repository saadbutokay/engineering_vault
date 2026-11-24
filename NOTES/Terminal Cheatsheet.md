```bash
# Navigation
# Where am I right now?
pwd
# Output: /home/yourname/documents

# List files and folders here
ls
# Output: projects/  downloads/  notes.txt

# List with  details
ls -l

# List with hidden details
ls -la

# Go into a folder
cd projects

# Go back one level
cd ..

# Go back two levels
cd ../..

# Go to home directory
cd ~

# Go to exact path
cd /home/yourname/documents/myproject
```

```bash
# Files & Folders
# Create a folder
mkdir my_project

# Create a file
touch main.py

# Create nested folders
mkdir -p projects/backend/myapp

# Copy a file
cp main.py main_backup.py

# Copy a folder
cp -R folder1_name folder2_name

# Rename/Move a file
mv main.py app.py

# Rename/Move a folder
mv folder1_name folder_new_name

# Delete a file (careful — no trash, permanent)
rm main.py

# Delete a folder and everything in it (BE CAREFUL)
rm -r my_folder/

# Print file contents
cat main.py

# Print first 10 lines
head main.py

# Print last 10 lines
tail main.py
```

```bash
# Running Things
# Run a Python file
python3 main.py

# Install a package
pip install requests

# See installed packages
pip list

# Clear the terminal screen
clear
```