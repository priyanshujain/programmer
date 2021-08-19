We follow a monthly release schedule. To release the version of the software, use the following guide.

## Github release process


### 1. Pull the latest changes

Pull the latest `develop` branch and master branch in your local machine.

```bash
git checkout master
git pull
git checkout develop
git pull
git fetch --tags
```

### 2. Commit new version

- Bump up the version numbers in the following files.

  - `config/app.yml`
  - `package.json`

- Commit the changes to develop branch with a title `Bump version to #{version_number}`



### 3. Prepare release

```bash
git flow release start '1.12.0' # replace your version number
git flow release finish '1.12.0' # replace your version number

# Leave default merge messages as it is
# prepare the tag message as 'v1.12.0' # replace your version number
```

### 4. Reverse merge master

Reverse merge master branch to develop branch.


```bash
git checkout develop
git merge master
```

### 5. Push changes to remote

Push master branch, develop branch and the tags.

```bash
# Push develop
git checkout develop
git push --no-verify

# Push master
git checkout master
git push --no-verify

# Push tags
git checkout develop
git push --tags --no-verify
```

### 6. Prepare release notes

- Compare the current version to the previous version using the tag compare feature in Github.
- Create a new release on Github from existing tags and update release notes.
- Ensure that the milestones exist for the subsequent 2 versions.
- Close the current milestone and move issues to the next milestone.

**Release note format**:

```
- Describe the changes in the current release
- Gratitudes towards those who have contributed to the project.
```


