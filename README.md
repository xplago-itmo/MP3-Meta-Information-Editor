# MP3 Meta Information Editor

The program allows you to get and set the meta-information of the `ID3v2` version (mainly `ID3v2.3`) of an `MP3` file.
- `--show` parameter displays all meta information in `ID | size | content` format;
- `--get=<ID>` parameter allows you to get information by frame ID;
- `--set<ID> --value<CONTENT>` allows you to set the CONTENT information in 8-byte encoding to the frame with the given ID.

## Used libraries

## Authors

- [@xplago-itmo](https://www.github.com/xplago-itmo)

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <stdlib.h>
```

## Used definitions

New types are used to simplify unsigned numbers:

```c
typedef uint32_t DWORD;
typedef uint16_t WORD;
typedef uint8_t BYTE;
```

## Frame of MP3 File Meta Tag
A frame is a structure, of which the main structure of the meta-information of an `MP3` file consists. The frame consists of 2 elements:
- header
- data (frame body)
The frame header consists of:
- `id: char[4]` - frame id of length 4 in string representation;
- `size: unsigned __int32` - size of frame data;
- `flags: WORD` - 16 flags;
```c
#pragma scalar_storage_order big-endian
struct IDv3_2_3_frame_header {
    char id[4];
    unsigned __int32 size;
    WORD flags;
};

#pragma scalar_storage_order little-endian
struct IDv3_2_3_frame {
    struct IDv3_2_3_frame_header header;
    char * data;
};
```
### Functions

Functions to make working with frames easier:
- `eq_frame_id(first: * char, second: * char): int` - check for equals two frame ids;
- `print_frame_data(id: * char, data: * char, size: unsigned int): void` - outputs data given the encoding;
- `print_frame_data(id: * char, data: char, size: unsigned int): void` - outputs data given the encoding;

```c
int eq_frame_id(char * first, char * second) {
    if (first[0] != second[0]) return 0;
    if (first[1] != second[1]) return 0;
    if (first[2] != second[2]) return 0;
    if (first[3] != second[3]) return 0;
    return 1;
}

void print_frame_data(char * id, char * data, unsigned int size) {
    if (strcmp(id, "APIC") == 0) {
        printf("image");
    } else if (
            (unsigned __int8) data[0] == 0x01
            && (unsigned __int8) data[1] == 0xFF
            && (unsigned __int8) data[2] == 0xFE
            ) {
        for (unsigned int i = 3; i < size; i++) {
            printf("%C", *((wchar_t *) &data[i]));
        }
    } else if (data[size] == '\0') {
        int j = 0;
        if (data[0] == '\0') j = 1;
        for (; j < size; j++) {
            if ((unsigned __int8) data[j] == 0xFF && (unsigned __int8) data[j + 1] == 0xFE) {
                printf("%ls", (wchar_t *) &data[j + 2]);
                j += wcslen((wchar_t *) &data[j + 2]) * 2;
            } else {
                printf("%c", *(data + j));
            }
        }
    } else {
        for (int j = 0; j < size; j++) {
            printf("%c", data[j]);
        };
    }
}

void print_frame_data_by_frame(struct IDv3_2_3_frame * frame) {
    print_frame_data(frame->header.id, frame->data, frame->header.size);
}
```

#### Read and Write Frame

- `read_frame(file: * FILE): struct IDv3_2_3_frame` - returns a frame structure read from the given file;
- `write_frame(frame: * struct IDv3_2_3_frame, file: * FILE): int` - write the given frame to the given file;
```c
struct IDv3_2_3_frame read_frame(FILE * file) {
    struct IDv3_2_3_frame_header header = {};
    fread(&header, 10, 1, file);
    char * data = calloc(header.size, 1);
    fread(data, header.size, 1, file);
    struct IDv3_2_3_frame frame = {};
    frame.header = header;
    frame.data = data;
    return frame;
}

int write_frame(struct IDv3_2_3_frame * frame, FILE * file) {
    fwrite(&frame->header, 10, 1, file);
    fwrite(frame->data, frame->header.size, 1, file);
    return 0;
}
```

## Tag of MP3 File
The tag is the main meta information structure of an `MP3` file. Consists of:
- header
- extended header (optional)
- frames
The tag header consists of:

- `id: char[3]` - constant value (market) `ID3`;
- `version_1: unsigned __int8` + `version_2: unsigned __int8` - represent the version of the `MP3` file;
- `flags: BYTE` - 8 flags (first of them depends on extended header availability;
- `raw_tag_size: unsigned __int8[4]` - summarize size of tag frames: 4 bytes of `b'0xxxxxxx` format;
- `frames: * IDv3_2_3_frame` - set of tag frames;

The tag extended header consists of:

- `header_size: unsigned __int32` - size of extended header (6 / 10 bytes);
- `flags: WORD` - 16 flags;
- `total_frame_crc: unsigned __int32` - optional CRC-32 data;

The complexity of the composition of the implementation of the tag in the form of a structure lies in the variability of the presence of an extended file header and the presence of crc-32 data in it.

```c
#pragma (push, 1)
struct IDv3_2_3_tag_header {
    char id[3]; // ID3
    unsigned __int8 version_1;
    unsigned __int8 version_2;
    BYTE flags;
    unsigned __int8 raw_tag_size[4];
};
#pragma (pop)

#pragma (push, 1)
struct IDv3_2_3_tag_extended_pre_header {
    unsigned __int32 header_size;
    WORD flags;
    unsigned __int32 padding_size;
};
#pragma (pop)

#pragma (push, 1)
struct IDv3_2_3_tag_extended_header {
    char available;
    struct IDv3_2_3_tag_extended_pre_header * pre_header;
    unsigned __int32 total_frame_crc;
};
#pragma (pop)

struct IDv3_2_3_tag {
    struct IDv3_2_3_tag_header header;
    struct IDv3_2_3_tag_extended_header extended_header;
    struct IDv3_2_3_frame * frames;
    char success;
    int count_of_frames;
};
```
### Functions
Functions that make it easier to work with the `MP3` file tag:
- `get_tag_size(header * struct IDv3_2_3_tag_header): unsigned __int32` - return header size turned from raw representation to using;
- `set_tag_size(struct IDv3_2_3_tag_header * header, unsigned __int32 size)` - set header size from using format to raw representation;
- `get_tag_header_flag(struct IDv3_2_3_tag_header * header, unsigned __int8 position): unsigned __int8` - return header flag by the given position;

```c
unsigned __int32 get_tag_size(struct IDv3_2_3_tag_header * header) {
    unsigned int res = 0;
    for (unsigned __int32 i = 0; i < 4; i++)
        res = res | (header->raw_tag_size[i] << (7 * (3 - i)));
    return res;
}

void set_tag_size(struct IDv3_2_3_tag_header * header, unsigned __int32 size) {
    for (unsigned int i = 0; i < 4; i++)
        header->raw_tag_size[3 - i] = (size >> (7 * i)) & 0x7F;
}

unsigned __int8 get_tag_header_flag(struct IDv3_2_3_tag_header * header, unsigned __int8 position) {
    return ((header->flags) >> (7 - position)) & 0b00000001;
}
```

#### Read and Write Tag
- `read_tag(FILE * file): struct IDv3_2_3_tag` - read tag from file and return it. The function can be divided into 3 parts:
    - header reading;
    - extended header reading;
    - iterative reading of frames, using a `read_frame` function call;
- `write_tag(tag: * struct IDv3_2_3_tag, file: * FILE): int` - write tag to file. The function can be divided into 3 parts:
    - header writing;
    - extended header writing;
    - iterative writing of frames, using a `write_frame` function call;

```c
struct IDv3_2_3_tag read_tag(FILE * file) {
    struct IDv3_2_3_tag tag = {{}, {}, NULL, 1};

    // read header
    struct IDv3_2_3_tag_header tag_header = {};
    fread(&tag_header, 10, 1, file);
    if (tag_header.id[0] != 'I' || tag_header.id[1] != 'D' || tag_header.id[2] != '3') {
        tag.success = 0;
        return tag;
    }
    tag.header = tag_header;

    // read extend header
    struct IDv3_2_3_tag_extended_header extended_header = {0, NULL, 0};
    if (get_tag_header_flag(&tag_header, 1) == 1) {
        struct IDv3_2_3_tag_extended_pre_header pre_header = {};
        fread(&pre_header, 6, 1, file);
        __int32 total_frame_crc = 0;
        if (pre_header.flags >> 15 == 1) {
            fread(&total_frame_crc, 4, 1, file);
        }
        extended_header.available = 1;
        extended_header.pre_header = &pre_header;
        extended_header.total_frame_crc = total_frame_crc;
    }
    tag.extended_header = extended_header;

    // read frames

    unsigned __int32 available_size = get_tag_size(&tag.header) - (tag.extended_header.available == 1 ? tag.extended_header.pre_header->header_size : 0);
    unsigned __int32 frame_size;
    tag.frames = (struct IDv3_2_3_frame *) calloc(1, available_size);
    int count_of_frames = 0;
    while (available_size > 0) {
        struct IDv3_2_3_frame frame = read_frame(file);
        if (frame.header.id[0] == 0 || frame.header.id[1] == 0 || frame.header.id[2] == 0 || frame.header.id[3] == 0) {
            fseek(file, -10, SEEK_CUR);
            break;
        }
        frame_size = 10 + frame.header.size;
        available_size -= frame_size;
        tag.frames[count_of_frames] = frame;
        count_of_frames++;
    }
    tag.count_of_frames = count_of_frames;
    return tag;
}

int write_tag(struct IDv3_2_3_tag * tag, FILE * file) {
    fwrite(&tag->header, 10, 1, file);
    if  (tag->extended_header.available == 1) {
        fwrite(tag->extended_header.pre_header, 10, 1, file);
        if (tag->extended_header.pre_header->flags >> 15 == 1) {
            fwrite(&tag->extended_header.total_frame_crc, 4, 1, file);
        }
    }
    for (int i = 0; i < tag->count_of_frames; i++) {
        write_frame(&tag->frames[i], file);
    }
    return 0;
}
```

#### Update Tag

`update_tag_by_name(tag: * struct IDv3_2_3_tag, data: * char, data_size: unsigned __int32, name: * char): void` - updates the tag structure by passing `data` to the frame named `name`,  or by creating a new frame with a `name` and `data`.

```c
void update_tag_by_name(struct IDv3_2_3_tag * tag, char * data, unsigned __int32 data_size, char * name) {
    struct IDv3_2_3_frame * frame = NULL;
    for (int i = 0; i < tag->count_of_frames; i++) {
        if (eq_frame_id((char *) &(tag->frames[i].header.id), name) == 1) {
            frame = &tag->frames[i];
        }
    }
    if (frame == NULL) {
        struct IDv3_2_3_frame_header header = {
                name[0],name[1],name[2],name[3],
                data_size,
                0
        };
        struct IDv3_2_3_frame new_frame = {header, data};
        tag->frames = realloc(tag->frames, (sizeof new_frame) * (tag->count_of_frames + 1));
        tag->count_of_frames++;
        tag->frames[tag->count_of_frames - 1] = new_frame;
        set_tag_size(&tag->header, get_tag_size(&tag->header) + data_size);
        printf("created %s with data: ", header.id);
        print_frame_data(name, data, data_size);
    } else {
        printf("%s: ", frame->header.id);
        print_frame_data_by_frame(frame);
        unsigned __int32 old_size = frame->header.size;
        frame->header.size = data_size;
        frame->data = data;
        set_tag_size(&tag->header, get_tag_size(&tag->header) + data_size - old_size);
        printf(" -> ");
        print_frame_data_by_frame(frame);
    }
    return 0;
}
```

## Program

### Tools

Auxiliary functions make it easier to work with MP3 files.

```c
#pragma scalar_storage_order big-endian
struct buffer {
    char buffer[4];
};

int ends_with_mp3(char * string) {
    string = strrchr(string, '.');
    if( string != NULL ) return(strcmp(string, ".mp3"));
    return(-1);
}

int check_frame_id_len(char * string) {
    if (strlen(string) == 4) return 1;
    return 0;
}
```

### Main Function

The main function of the program can be divided into several stages:

- handles command line arguments;
- opens a `MP3` file and reads the meta information tag;
- if there are `--set` and `--value` options, it resets the meta-information of the file:
  - updates the tag structure
  - writes the rest of the information to a temporary file (so as not to load RAM);
  - writes a new tag to the cleaned up old file;
  - writes information from the temporary file to the old file;
- if there is a `--get` parameter, then it looks for a frame in the tag with the given name and displays the frame data;
- if there is a `--show` parameter, then it displays information about all frames of the tag one by one.

```c
int main(int argc, char *argv[]) {
    char * filename = NULL;
    char * set_frame_id = NULL;
    char * get_frame_id = NULL;
    char * value = NULL;
    int show_all = 0;

    char has_error = 0;
    for (int i = 1; i < argc; i++) {
        if (strcmp(argv[i], "--show") == 0) show_all = 1;

        char * parameter = strtok(argv[i], "=");
        if (strcmp(parameter, "--filepath") == 0) {
            filename = strtok(NULL, "\0");
            if (ends_with_mp3(filename) != 0) {
                fprintf(stderr, "Error: Invalid input file name \"%s\"\n", filename);
                has_error = 1;
            }
        }

        if (strcmp(parameter, "--set") == 0) {
            set_frame_id = strtok(NULL, "\0");
            if (check_frame_id_len(set_frame_id) == 0) {
                fprintf(stderr, "Error: Invalid frame id \"%s\"\n", set_frame_id);
                has_error = 1;
            }
        }

        if (strcmp(parameter, "--get") == 0) {
            get_frame_id = strtok(NULL, "\0");
            if (check_frame_id_len(get_frame_id) == 0) {
                fprintf(stderr, "Error: Invalid frame id \"%s\"\n", get_frame_id);
                has_error = 1;
            }
        }

        if (strcmp(parameter, "--value") == 0) {
            value = strtok(NULL, "\0");
        }
    }

    if (filename == NULL) {
        fprintf(stderr, "Error: Missing required parameter --filepath\n");
        has_error = 1;
    } else {
        char * filename_tmp = (char *) calloc(strlen(filename), 1);
        strcpy(filename_tmp, filename);

        FILE * file = fopen(filename, "rb");

        if (file == NULL) {
            fprintf(stderr, "Error: File not exists\n");
            return -1;
        }

        struct IDv3_2_3_tag tag = read_tag(file);
        if (set_frame_id != NULL) {
            if (value == NULL) {
                fprintf(stderr, "Error: Missing required parameter --value with parameter --set\n");
                return -1;
            } else {
                FILE * tmp_file = fopen(strtok(filename, "."), "wb");
                char * data = (char *) calloc(strlen(value) + 1, 1);
                data[0] = '\0';
                printf("%c", data[0]);
                strcat(data + 1, value);
                update_tag_by_name(&tag, data, strlen(value) + 1, set_frame_id);

                struct buffer buffer = {{0, 0, 0, 0}};
                unsigned long long res1 = 1;
                unsigned long long res2 = 1;

                while (res1 > 0 && res2 > 0) {
                    res1 = fread(&buffer, sizeof buffer, 1, file);
                    if (res1 > 0) res2 = fwrite(&buffer, sizeof buffer, 1, tmp_file);
                }
                fclose(file);
                fclose(tmp_file);

                FILE * file_n = fopen(filename_tmp, "wb");
                FILE * tmp_file_n = fopen(strtok(filename_tmp, "."), "rb");

                write_tag(&tag, file_n);
                for (int i = 0; i < sizeof buffer; i++) buffer.buffer[i] = 0;
                res1 = 1;
                res2 = 1;
                while (res1 > 0 && res2 > 0) {
                    res1 = fread(&buffer, 1, sizeof buffer, tmp_file_n);
                    if (res1 > 0) res2 = fwrite(&buffer, 1, res1, file_n);
                }

                fclose(file_n);
                fclose(tmp_file_n);
                remove(filename);
            }
        }
        
        if (get_frame_id != NULL) {
            struct IDv3_2_3_frame * frame = NULL;
            for (int i = 0; i < tag.count_of_frames; i++) {
                if (eq_frame_id((char *) &(tag.frames[i].header.id), get_frame_id) == 1) {
                    frame = &tag.frames[i];
                }
            }
            if (frame != NULL) {
                print_frame_data_by_frame(frame);
            } else {
                printf("No frame found");
            }
        }

        if (show_all == 1) {
            printf("ID   | size \t| data\n");
            for (int i = 0; i < tag.count_of_frames; i++) {
                printf("%s | ", tag.frames[i].header.id);
                printf("%d \t| ", tag.frames[i].header.size);
                print_frame_data_by_frame(&tag.frames[i]);
                printf("\n");
            }
        }
    }

    if (has_error == 1) {
        return -1;
    }

    return 0;
}
```

## Full Code
```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <stdlib.h>

typedef uint32_t DWORD;
typedef uint16_t WORD;
typedef uint8_t BYTE;

// IDv3_2_3_frame

#pragma scalar_storage_order big-endian
struct IDv3_2_3_frame_header {
    char id[4];
    unsigned __int32 size;
    WORD flags;
};

#pragma scalar_storage_order little-endian
struct IDv3_2_3_frame {
    struct IDv3_2_3_frame_header header;
    char * data;
};

int eq_frame_id(char * first, char * second) {
    if (first[0] != second[0]) return 0;
    if (first[1] != second[1]) return 0;
    if (first[2] != second[2]) return 0;
    if (first[3] != second[3]) return 0;
    return 1;
}

struct IDv3_2_3_frame read_frame(FILE * file) {
    struct IDv3_2_3_frame_header header = {};
    fread(&header, 10, 1, file);
    char * data = calloc(header.size, 1);
    fread(data, header.size, 1, file);
    struct IDv3_2_3_frame frame = {};
    frame.header = header;
    frame.data = data;
    return frame;
}

int write_frame(struct IDv3_2_3_frame * frame, FILE * file) {
    fwrite(&frame->header, 10, 1, file);
    fwrite(frame->data, frame->header.size, 1, file);
    return 0;
}

// IDv3_2_3_tag

#pragma (push, 1)
struct IDv3_2_3_tag_header {
    char id[3]; // ID3
    unsigned __int8 version_1;
    unsigned __int8 version_2;
    BYTE flags;
    unsigned __int8 raw_tag_size[4];
};
#pragma (pop)

#pragma (push, 1)
struct IDv3_2_3_tag_extended_pre_header {
    unsigned __int32 header_size;
    WORD flags;
    unsigned __int32 padding_size;
};
#pragma (pop)

#pragma (push, 1)
struct IDv3_2_3_tag_extended_header {
    char available;
    struct IDv3_2_3_tag_extended_pre_header * pre_header;
    unsigned __int32 total_frame_crc;
};
#pragma (pop)

struct IDv3_2_3_tag {
    struct IDv3_2_3_tag_header header;
    struct IDv3_2_3_tag_extended_header extended_header;
    struct IDv3_2_3_frame * frames;
    char success;
    int count_of_frames;
};

unsigned __int32 get_tag_size(struct IDv3_2_3_tag_header * header) {
    unsigned int res = 0;
    for (unsigned __int32 i = 0; i < 4; i++)
        res = res | (header->raw_tag_size[i] << (7 * (3 - i)));
    return res;
}

void set_tag_size(struct IDv3_2_3_tag_header * header, unsigned __int32 size) {
    for (unsigned int i = 0; i < 4; i++)
        header->raw_tag_size[3 - i] = (size >> (7 * i)) & 0x7F;
}

unsigned __int8 get_tag_header_flag(struct IDv3_2_3_tag_header * header, unsigned __int8 position) {
    return ((header->flags) >> (7 - position)) & 0b00000001;
}

struct IDv3_2_3_tag read_tag(FILE * file){
    struct IDv3_2_3_tag tag = {{}, {}, NULL, 1};

    // read header
    struct IDv3_2_3_tag_header tag_header = {};
    fread(&tag_header, 10, 1, file);
    if (tag_header.id[0] != 'I' || tag_header.id[1] != 'D' || tag_header.id[2] != '3') {
        tag.success = 0;
        return tag;
    }
    tag.header = tag_header;

    // read extend header
    struct IDv3_2_3_tag_extended_header extended_header = {0, NULL, 0};
    if (get_tag_header_flag(&tag_header, 1) == 1) {
        struct IDv3_2_3_tag_extended_pre_header pre_header = {};
        fread(&pre_header, 6, 1, file);
        __int32 total_frame_crc = 0;
        if (pre_header.flags >> 15 == 1) {
            fread(&total_frame_crc, 4, 1, file);
        }
        extended_header.available = 1;
        extended_header.pre_header = &pre_header;
        extended_header.total_frame_crc = total_frame_crc;
    }
    tag.extended_header = extended_header;

    // read frames

    unsigned __int32 available_size = get_tag_size(&tag.header) - (tag.extended_header.available == 1 ? tag.extended_header.pre_header->header_size : 0);
    unsigned __int32 frame_size;
    tag.frames = (struct IDv3_2_3_frame *) calloc(1, available_size);
    int count_of_frames = 0;
    while (available_size > 0) {
        struct IDv3_2_3_frame frame = read_frame(file);
        if (frame.header.id[0] == 0 || frame.header.id[1] == 0 || frame.header.id[2] == 0 || frame.header.id[3] == 0) {
            fseek(file, -10, SEEK_CUR);
            break;
        }
        frame_size = 10 + frame.header.size;
        available_size -= frame_size;
        tag.frames[count_of_frames] = frame;
        count_of_frames++;
    }
    tag.count_of_frames = count_of_frames;
    return tag;
}

int write_tag(struct IDv3_2_3_tag * tag, FILE * file) {
    fwrite(&tag->header, 10, 1, file);
    if  (tag->extended_header.available == 1) {
        fwrite(tag->extended_header.pre_header, 10, 1, file);
        if (tag->extended_header.pre_header->flags >> 15 == 1) {
            fwrite(&tag->extended_header.total_frame_crc, 4, 1, file);
        }
    }
    for (int i = 0; i < tag->count_of_frames; i++) {
        write_frame(&tag->frames[i], file);
    }
    return 0;
}

void print_frame_data(char * id, char * data, unsigned int size) {
    if (strcmp(id, "APIC") == 0) {
        printf("image");
    } else if (
            (unsigned __int8) data[0] == 0x01
            && (unsigned __int8) data[1] == 0xFF
            && (unsigned __int8) data[2] == 0xFE
            ) {
        for (unsigned int i = 3; i < size; i++) {
            printf("%C", *((wchar_t *) &data[i]));
        }
    } else if (data[size] == '\0') {
        int j = 0;
        if (data[0] == '\0') j = 1;
        for (; j < size; j++) {
            if ((unsigned __int8) data[j] == 0xFF && (unsigned __int8) data[j + 1] == 0xFE) {
                printf("%ls", (wchar_t *) &data[j + 2]);
                j += wcslen((wchar_t *) &data[j + 2]) * 2;
            } else {
                printf("%c", *(data + j));
            }
        }
    } else {
        for (int j = 0; j < size; j++) {
            printf("%c", data[j]);
        };
    }
}

void print_frame_data_by_frame(struct IDv3_2_3_frame * frame) {
    print_frame_data(frame->header.id, frame->data, frame->header.size);
}

char * update_tag_by_name(struct IDv3_2_3_tag * tag, char * data, unsigned __int32 data_size, char * name) {
    struct IDv3_2_3_frame * frame = NULL;
    for (int i = 0; i < tag->count_of_frames; i++) {
        if (eq_frame_id((char *) &(tag->frames[i].header.id), name) == 1) {
            frame = &tag->frames[i];
        }
    }
    if (frame == NULL) {
        struct IDv3_2_3_frame_header header = {
                name[0],name[1],name[2],name[3],
                data_size,
                0
        };
        struct IDv3_2_3_frame new_frame = {header, data};
        tag->frames = realloc(tag->frames, (sizeof new_frame) * (tag->count_of_frames + 1));
        tag->count_of_frames++;
        tag->frames[tag->count_of_frames - 1] = new_frame;
        set_tag_size(&tag->header, get_tag_size(&tag->header) + data_size);
        printf("created %s with data: ", header.id);
        print_frame_data(name, data, data_size);
    } else {
        printf("%s: ", frame->header.id);
        print_frame_data_by_frame(frame);
        unsigned __int32 old_size = frame->header.size;
        frame->header.size = data_size;
        frame->data = data;
        set_tag_size(&tag->header, get_tag_size(&tag->header) + data_size - old_size);
        printf(" -> ");
        print_frame_data_by_frame(frame);
    }
    return 0;
}

#pragma scalar_storage_order big-endian
struct buffer {
    char buffer[4];
};

int ends_with_mp3(char * string) {
    string = strrchr(string, '.');
    if( string != NULL ) return(strcmp(string, ".mp3"));
    return(-1);
}

int check_frame_id_len(char * string) {
    if (strlen(string) == 4) return 1;
    return 0;
}

int main(int argc, char *argv[]) {
    char * filename = NULL;
    char * set_frame_id = NULL;
    char * get_frame_id = NULL;
    char * value = NULL;
    int show_all = 0;

    char has_error = 0;
    for (int i = 1; i < argc; i++) {
        if (strcmp(argv[i], "--show") == 0) show_all = 1;

        char * parameter = strtok(argv[i], "=");
        if (strcmp(parameter, "--filepath") == 0) {
            filename = strtok(NULL, "\0");
            if (ends_with_mp3(filename) != 0) {
                fprintf(stderr, "Error: Invalid input file name \"%s\"\n", filename);
                has_error = 1;
            }
        }

        if (strcmp(parameter, "--set") == 0) {
            set_frame_id = strtok(NULL, "\0");
            if (check_frame_id_len(set_frame_id) == 0) {
                fprintf(stderr, "Error: Invalid frame id \"%s\"\n", set_frame_id);
                has_error = 1;
            }
        }

        if (strcmp(parameter, "--get") == 0) {
            get_frame_id = strtok(NULL, "\0");
            if (check_frame_id_len(get_frame_id) == 0) {
                fprintf(stderr, "Error: Invalid frame id \"%s\"\n", get_frame_id);
                has_error = 1;
            }
        }

        if (strcmp(parameter, "--value") == 0) {
            value = strtok(NULL, "\0");
        }
    }

    if (filename == NULL) {
        fprintf(stderr, "Error: Missing required parameter --filepath\n");
        has_error = 1;
    } else {
        char * filename_tmp = (char *) calloc(strlen(filename), 1);
        strcpy(filename_tmp, filename);

        FILE * file = fopen(filename, "rb");

        if (file == NULL) {
            fprintf(stderr, "Error: File not exists\n");
            return -1;
        }

        struct IDv3_2_3_tag tag = read_tag(file);
        if (set_frame_id != NULL) {
            if (value == NULL) {
                fprintf(stderr, "Error: Missing required parameter --value with parameter --set\n");
                return -1;
            } else {
                FILE * tmp_file = fopen(strtok(filename, "."), "wb");
                char * data = (char *) calloc(strlen(value) + 1, 1);
                data[0] = '\0';
                printf("%c", data[0]);
                strcat(data + 1, value);
                update_tag_by_name(&tag, data, strlen(value) + 1, set_frame_id);

                struct buffer buffer = {{0, 0, 0, 0}};
                unsigned long long res1 = 1;
                unsigned long long res2 = 1;

                while (res1 > 0 && res2 > 0) {
                    res1 = fread(&buffer, sizeof buffer, 1, file);
                    if (res1 > 0) res2 = fwrite(&buffer, sizeof buffer, 1, tmp_file);
                }
                fclose(file);
                fclose(tmp_file);

                FILE * file_n = fopen(filename_tmp, "wb");
                FILE * tmp_file_n = fopen(strtok(filename_tmp, "."), "rb");

                write_tag(&tag, file_n);
                for (int i = 0; i < sizeof buffer; i++) buffer.buffer[i] = 0;
                res1 = 1;
                res2 = 1;
                while (res1 > 0 && res2 > 0) {
                    res1 = fread(&buffer, 1, sizeof buffer, tmp_file_n);
                    if (res1 > 0) res2 = fwrite(&buffer, 1, res1, file_n);
                }

                fclose(file_n);
                fclose(tmp_file_n);
                remove(filename);
            }
        }
        if (get_frame_id != NULL) {
            struct IDv3_2_3_frame * frame = NULL;
            for (int i = 0; i < tag.count_of_frames; i++) {
                if (eq_frame_id((char *) &(tag.frames[i].header.id), get_frame_id) == 1) {
                    frame = &tag.frames[i];
                }
            }
            if (frame != NULL) {
                print_frame_data_by_frame(frame);
            } else {
                printf("No frame found");
            }
        }

        if (show_all == 1) {
            printf("ID   | size \t| data\n");
            for (int i = 0; i < tag.count_of_frames; i++) {
                printf("%s | ", tag.frames[i].header.id);
                printf("%d \t| ", tag.frames[i].header.size);
                print_frame_data_by_frame(&tag.frames[i]);
                printf("\n");
            }
        }
    }

    if (has_error == 1) {
        return -1;
    }

    return 0;
}
```