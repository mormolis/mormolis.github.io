---
layout: post
category: programming
---

In Junit4 we had TemporaryFolder. In Junit5 the way to test objects that are interacting with the file system it is pretty straightforward with the use of `@TempDir`.

To make things easier to understand lets assume that we have a class that reads all the files, directories, sub directories and files in sub directories and gathers information about them in a form of a simple dto that contains the extension, if it is directory, the absolute path etc. and adds them in a list.

So given a Path you get a list of objects that contain the file information.

In order to test this functionality we are going to use the TempDir helper from Junit5.
This will create a temporary directory in our filesystem that will be cleaned automatically when our testing is finished.

Check out the example bellow

```java
package directory.scanner.attributes.reader;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class FileAttributeDtoReaderTest {

    FileAttributeReader fileAttributeReader = new FileAttributeReader();

    @Test
    void returnsAListOfAllFilesAndDirectoriesInTheFileTree(@TempDir Path tempDirectory) throws IOException {

        createFilesAndDirectoriesInTemporaryDirectory(tempDirectory);

        List<AttributeDto> actual = fileAttributeReader.scanFilesInADirectory(tempDirectory);

        assertThat(actual).contains(expectedListOfFileAttributes(tempDirectory));
    }

    private void createFilesAndDirectoriesInTemporaryDirectory(Path tempDirectory) throws IOException {
        Files.createFile(tempDirectory.resolve("file1.txt"));
        Files.createFile(tempDirectory.resolve("file2."));
        Files.createFile(tempDirectory.resolve("file3.txt"));
        Files.createFile(tempDirectory.resolve("file4"));
        Path dir = Files.createDirectory(tempDirectory.resolve("dir"));
        Files.createFile(dir.resolve("fileInDirectory"));
    }

    private AttributeDto[] expectedListOfFileAttributes(Path tempDirectory) {

        AttributeDto attributeDto0 = new AttributeDto(tempDirectory.toAbsolutePath().toString(), "", 4096, true);
        AttributeDto attributeDto1 = new AttributeDto(tempDirectory.toAbsolutePath() + File.separator + "file1.txt", "txt", 0, false);
        AttributeDto attributeDto2 = new AttributeDto(tempDirectory.toAbsolutePath() + File.separator + "file2.", "", 0, false);
        AttributeDto attributeDto3 = new AttributeDto(tempDirectory.toAbsolutePath() + File.separator + "file3.txt", "txt", 0, false);
        AttributeDto attributeDto4 = new AttributeDto(tempDirectory.toAbsolutePath() + File.separator + "file4", "", 0, false);
        AttributeDto attributeDto5 = new AttributeDto(tempDirectory.toAbsolutePath() + File.separator + "dir", "", 4096, true);
        AttributeDto attributeDto6 = new AttributeDto(tempDirectory.toAbsolutePath() + File.separator + "dir" + File.separator + "fileInDirectory", "", 0, false);
        AttributeDto[] expected = new AttributeDto[7];
        expected[0] = attributeDto0;
        expected[1] = attributeDto1;
        expected[2] = attributeDto2;
        expected[3] = attributeDto3;
        expected[4] = attributeDto4;
        expected[5] = attributeDto5;
        expected[6] = attributeDto6;
        return expected;
    }
}
```

Check on line 20 how we pass the temporary directory as a test parameter annotated accordingly.

`@TempDir` will create a directory (depending on your OS) on a tmp folder like `/tmp/junit6110933399026626368`, but this should not worry us since junit is going to get rid of it as soon as we finish with our test.

Method `createFilesAndDirectoriesInTemporaryDirectory` is using the temporary directory along with Files api (java 7) to create folder structure like the following.

```
tempDir/

  file1.txt
  file2.
  file3.txt
  file4

  dir/
    fileInDirectory
```

Finally the method on line 38 created the expected result. On the `AtributeDto` we can spot the absolute path of our generated files being passed as the first parameter. Second parameter is the extension of the file, third is the size and finally a boolean that dictates whether the file is a directory or not.

On this example for the assertions I am using a great assertion library which is called assertJ and can be found [here](https://assertj.github.io/doc/).
  
  
