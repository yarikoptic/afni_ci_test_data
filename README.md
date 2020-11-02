# Test data management

This repository stores the data required for AFNI's continuous integration testing on CircleCI (and testing run locally with the run_afni_tests.py tool). If you are not using this tool but still wish to use data from this repository you must [install datalad](https://www.datalad.org/get_datalad.htmlhttps://www.datalad.org/get_datalad.html) if you have not already.

To get the repository:

```
datalad install https://github.com/afni/afni_ci_test_data.git
```


## Adding more data to the test data repository

N.B. You need to modify your data repository to have push access to the afni server to do this. See `git configuration` in the trouble shooting tips below.

### Adding arbitrary data

Examples of adding data to the repository can be seen in the setup script in this repository. Data can be extracted from archives at URLs, installed via datalad from other online repositories (OpenNeuro and the like), or created on the fly by executing an AFNI command that generates data that is then saved to the repository. For examples, read through the initial [setup of this repository](./script_for_repo_setup.sh)


### Updating sample test data

When new tests are added, or the expected output of a particular test changes you will need to add some more data to this repository. Follow these instructions to do this...


Be very careful that the behavior from your tested function and the test is what you want. This will create the new "correct" output. It is best to run all of the test commands inside the development container. This will reduce confusing results on circleci. The best way to start a container is to follow the instructions emitted when you attempt to run container testing in debug mode i.e. `run_afni_tests.py -d container`


1. Run all tests (they should all pass except the ones you wish to create data for)


```
cd /opt/afni/src/tests;                        \
./run_afni_tests.py                            \
    --build-dir /opt/afni/build                \
    -u                                         \
    -e='--runveryslow '                        \
    local
```

2. Rerun the failed tests in "create sample output" mode.


```
./run_afni_tests.py                            \
    --build-dir /opt/afni/build                \
    -u                                         \
    -e='--runveryslow --create_sample_output'  \
    --lf                                       \
    -v verbose                                 \
    local
```

3. Copy the sample data into the test data tree (you will need to modify the directory path below, be careful with trailing slashes):


```
rsync -aP                                                               \
    tests/sample_output_of_tests/sample_output_2020_10_28_180130/       \
    afni_ci_test_data/sample_test_output/sample_test_output/
```

4. Add them to the datalad repo


```
cd afni_ci_test_data
datalad save -m 'update sample output data'
```

5. Publish to the github and afni server:


```
datalad publish --transfer-data=all --to=origin
```

Note: the afni server has to container the binary blobs and have the binary blobs globaly accessible. The git repository doesn't need to look up to data, so for example it is fine to have an old version of master checked-out. The data will look out of data but it will be providing the appropriate files for download to datalad/git-annex.

6. Fix permissions on the server... I can't find a work around for this right now.

Fixing permissions:


```
find /fraid/pub/dist/data/afni_ci_test_data -type d  ! -perm -o=x -exec chmod o+x {} \;; find /fraid/pub/dist/data/afni_ci_test_data -type f  ! -perm -o=r -exec chmod o+r {} \;
```

7. Test that the data can be fetched:


```
cd /tmp
datalad install https://github.com/afni/afni_ci_test_data.git
cd afni_ci_test_data
# Insert your actual sample_test_output that you wish to test for
datalad get sample_test_output/3dTproject
```

If the above doesn't work start working through the trouble-shooting tips below. If it does then celebrate and do the dance of joy.


8. Update your reference to the data in the main code repository


```
cd ~/afni # where ever you keep your afni source code
git add tests/afni_ci_tests_data
git commit -m 'update data for tests'
```

## Troubleshooting tips:


### git configuration

The file ./git/config file in the afni_ci_test_data repository does not contain an entry that looks like the following you may have some issues:

```
[remote "afni_ci_test_data"]
    url = https://afni.nimh.nih.gov/pub/dist/data/afni_ci_test_data/.git
    pushurl = afni.nimh.nih.gov:/fraid/pub/dist/data/afni_ci_test_data
    fetch = +refs/heads/*:refs/remotes/afni_ci_test_data/*
    annex-bare = false
    annex-uuid = c1ce38d5-c2ef-48c6-a1f2-e207215d0717
```

Specifically, you should have a pushurl configured. If you do not, `datalad publish` will try and fail to write via https.

### Consulting the ghosts of the past

Check commits logs in the afni_ci_test_data repo, these provide an excellent guide to commands that were used to add data to the repository.

### Fixing browser access

Due to NIH server configuration constraint the files need to be indexed if you wish to access them directly via a browser. This could be implemented as a git hook that is executed on push if that is desirable. Overall I think having browser access is not required though (script stored in repository if you need it, add it to your path):


```
@make.directory.index -nested -dirs /fraid/pub/dist/data/afni_ci_test_data
```


### Permission fix attempt

Executed the following (oct 2020) to see if that lead to a better situation (I do not think it does sadly):


```
chgrp -R $(id -g) afni_ci_test_data
chmod -R g+swX afni_ci_test_data
chmod -R o+srX afni_ci_test_data
```

### annex remote accessibility

Sometimes when the remote is not accessible the annex remote will be disabled. This can be undone by modifying .git/config or using `git-annex enableremote afni_ci_test_data`


### Potential quick fix

Sometimes the quick and easy fix is to run datalad update
