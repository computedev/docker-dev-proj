# Check the hostname to prove we are in a container
hostname

# Clone the repository, this will fail
git clone https://github.com/spurin/cmatrix.git

# Prepare apk for use and install git
apk update
apk add git

# Clone the repository, will succeed
git clone https://github.com/spurin/cmatrix.git

# Change to the cmatrix source code and view the code
cd cmatrix/
ls -l

# Prepare compilation, will fail, missing autoconf
autoreconf -i

# Install autoconf
apk add autoconf

# Prepare compilation, will fail, missing automake
autoreconf -i

# Install automake
apk add automake

# Prepare compilation, will succeed, confirm with echo $?
autoreconf -i
echo $?

# Prepare configure, will fail, missing compiler
./configure LDFLAGS="-static"

# Install compiler
apk add alpine-sdk

# Prepare configure, will fail, missing dependencies
./configure LDFLAGS="-static"

# Install dependencies and create missing directories
apk add ncurses-dev ncurses-static
mkdir -p /usr/lib/kbd/consolefonts /usr/share/consolefonts

# Prepare configure, will succeed
./configure LDFLAGS="-static"

# Compile and view result
make
ls -l ./cmatrix

# Run cmatrix
./cmatrix