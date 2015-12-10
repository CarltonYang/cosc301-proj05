#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <assert.h>
#include <ctype.h>
#include "bootsect.h"
#include "bpb.h"
#include "direntry.h"
#include "fat.h"
#include "dos.h"
#define TOTAL_CLUSTERS(bpb) (bpb->bpbSectors / bpb->bpbSecPerClust)
#define CLUSTER_SIZE(bpb) (bpb->bpbSecPerClust * bpb->bpbBytesPerSec)
typedef struct node{
	uint16_t cluster;
	struct node *next;
} Node; 
void cluster_add(uint16_t cluster, Node **clusterlist);
void clusterlist_clear( Node *file_list);
int check_cluster(uint16_t cluster, Node *headoflist);
int fix_inconsistency(struct direntry* dirent,uint8_t *image_buf,
		      char* filename, struct bpb33* bpb, Node **clusterlist );
uint16_t get_dirent(struct direntry *dirent, char *buffer);
void write_dirent(struct direntry *dirent, char *filename, 
		  uint16_t start_cluster, uint32_t size);
void usage(char *progname) {
    fprintf(stderr, "usage: %s <imagename>\n", progname);
    exit(1);
}
void create_dirent(struct direntry *dirent, char *filename, 
		   uint16_t start_cluster, uint32_t size,
		   uint8_t *image_buf, struct bpb33* bpb);

void free_cluster(struct direntry *dirent, uint8_t *image_buf, struct bpb33* bpb, int size_by_dirent){
    uint16_t cluster = getushort(dirent->deStartCluster);
    uint16_t cluster_size = bpb->bpbBytesPerSec * bpb->bpbSecPerClust;

    uint16_t byte_count = 0;
    uint16_t prev_cluster = cluster;

    while (byte_count < size_by_dirent) {
        byte_count += cluster_size;
        prev_cluster = cluster;
        cluster = get_fat_entry(cluster, image_buf, bpb);
    }
    if (byte_count != 0) {
        set_fat_entry(prev_cluster, FAT12_MASK & CLUST_EOFS, image_buf, bpb);
    }
    // marking the other clusters pointed to by the FAT chain as free
    while (!is_end_of_file(cluster)) {
        
        uint16_t oldcluster = cluster;
        cluster = get_fat_entry(cluster, image_buf, bpb);

        set_fat_entry(oldcluster, FAT12_MASK & CLUST_FREE, image_buf, bpb);
    }
}

void follow_dir(uint16_t cluster, int indent,
		uint8_t *image_buf, struct bpb33* bpb, Node **clusterlist)
{
    while (is_valid_cluster(cluster, bpb))
    {
	cluster_add(cluster,clusterlist);
        struct direntry *dirent = (struct direntry*)cluster_to_addr(cluster, image_buf, bpb);
	char buffer[MAXFILENAME];
	
        int numDirEntries = (bpb->bpbBytesPerSec * bpb->bpbSecPerClust) / sizeof(struct direntry);
        int i = 0;
	for ( ; i < numDirEntries; i++)

            cluster_add(cluster,clusterlist);
            uint16_t followclust = get_dirent(dirent, buffer);
	    fix_inconsistency(dirent, image_buf, buffer, bpb, clusterlist);
            if (followclust)
                {follow_dir(followclust, indent+1, image_buf, bpb, clusterlist);}
            dirent++;
	
	cluster = get_fat_entry(cluster, image_buf, bpb);
    }
}

int cluster_size(struct direntry *dirent, uint8_t *image_buf, struct bpb33 *bpb, Node **cluster_list){
    uint16_t cluster = getushort(dirent->deStartCluster);
    uint16_t cluster_size = bpb->bpbBytesPerSec * bpb->bpbSecPerClust;

    int byte_count = 0;
    cluster_add(cluster, cluster_list);
    uint16_t prev_cluster = cluster;
    if (is_end_of_file(cluster)) {
        byte_count = 512;
    }
    while (!is_end_of_file(cluster) && cluster < 2849) {   

        if (cluster == (FAT12_MASK & CLUST_BAD)) {
            printf("Bad cluster: cluster number %d \n", cluster);
            set_fat_entry(prev_cluster, FAT12_MASK & CLUST_EOFS, image_buf, bpb);
            break;
        }

        if (cluster == (FAT12_MASK & CLUST_FREE)) {
            //printf("Free cluster in chain: cluster number %d \n", cluster); 
            set_fat_entry(prev_cluster, FAT12_MASK & CLUST_EOFS, image_buf, bpb);
            break;   
        }

        byte_count += cluster_size;
        prev_cluster = cluster;
        cluster = get_fat_entry(cluster, image_buf, bpb);
        
        if (prev_cluster == cluster) {
            printf("Cluster refers to itself! Setting it as end of file. \n");
            set_fat_entry(prev_cluster, FAT12_MASK & CLUST_EOFS, image_buf, bpb);
            break;   
        }

        cluster_add(cluster, cluster_list);
    }

    return byte_count;
}

int fix_inconsistency(struct direntry* dirent,uint8_t *image_buf,
		      char* filename, struct bpb33* bpb, Node **clusterlist ){
	uint32_t size_by_dirent = getulong(dirent->deFileSize);// find the file size according to directory entry
	int size_by_cluster = cluster_size(dirent, image_buf,bpb, clusterlist);//find the file size according to clusters
	
        if (size_by_cluster != 0 && size_by_dirent < size_by_cluster - 512 ) { // fix the FAT
            printf("Inconsistent file: %s (size in dir entry: %d, size in FAT chain: %d) \n", filename, size_by_dirent, size_by_cluster);
            free_cluster(dirent, image_buf, bpb, size_by_dirent);
           
        }

        else if (size_by_dirent > size_by_cluster) { //fix the dir entry
            printf("Inconsistent file: %s (size in dir entry: %d, size in FAT chain: %d) \n", filename, size_by_dirent, size_by_cluster);
            putulong(dirent->deFileSize, size_by_cluster);
           
        }
	return 1;
}

void traverse_root(uint8_t *image_buf, struct bpb33* bpb)
{
    Node* newlist=NULL;
    uint16_t cluster = 0;
    struct direntry *dirent = (struct direntry*)cluster_to_addr(cluster, image_buf, bpb);
    char buffer[MAXFILENAME];
    int i = 0;
    for ( ; i < bpb->bpbRootDirEnts; i++)
    {
        uint16_t followclust = get_dirent(dirent, buffer);
	if (dirent->deAttributes == ATTR_NORMAL) {
	    fix_inconsistency(dirent,  image_buf,buffer, bpb, &newlist);}
	cluster_add(followclust, &newlist);
        if (is_valid_cluster(followclust, bpb)) {
            cluster_add(followclust, &newlist);
            follow_dir(followclust, 1, image_buf, bpb, &newlist);
        }
        dirent++;
    }



}


int main(int argc, char** argv) {
    uint8_t *image_buf;
    int fd;
    struct bpb33* bpb;
    if (argc < 2) {
	usage(argv[0]);
    }

    image_buf = mmap_file(argv[1], &fd);
    bpb = check_bootsector(image_buf);

    // your code should start here...

    traverse_root(image_buf, bpb);



    unmmap_file(image_buf, &fd);
    return 0;
}










//Linked list functions
void cluster_add(uint16_t cluster, Node **file_list) {
    if (check_cluster(cluster, *file_list)) { // add smartly, don't add if it's already there.
        return;
    }
    Node *newnode= malloc(sizeof(Node));
    newnode->cluster = cluster;
    newnode->next = NULL;
    Node *curr = *file_list;
    if (curr == NULL) {
        *file_list = newnode;
        return;
    }
    while (curr->next != NULL) {
        curr = curr->next;
    }
    curr->next = newnode;
    newnode->next = NULL;
}

void clusterlist_clear( Node *file_list) {
    while (file_list != NULL) {
         Node *tmp = file_list;
        file_list = file_list->next;
        free(tmp);
    }
}

int check_cluster(uint16_t cluster, Node *headoflist)
{
    Node *current = headoflist;
    while (current != NULL) {
        if (cluster == current->cluster) {
            return 1;
        }
        current = current->next;
    }
    return 0;
}


// functions for inconsistency problems
uint16_t get_dirent(struct direntry *dirent, char *buffer)
{
    uint16_t followclust = 0;
    memset(buffer, 0, MAXFILENAME);

    int i;
    char name[9];
    char extension[4];
    uint16_t file_cluster;
    name[8] = ' ';
    extension[3] = ' ';
    memcpy(name, &(dirent->deName[0]), 8);
    memcpy(extension, dirent->deExtension, 3);
    if (name[0] == SLOT_EMPTY)
    {
	return followclust;
    }

    /* skip over deleted entries */
    if (((uint8_t)name[0]) == SLOT_DELETED)
    {
	return followclust;
    }

    if (((uint8_t)name[0]) == 0x2E)
    {
	// dot entry ("." or "..")
	// skip it
        return followclust;
    }

    /* names are space padded - remove the spaces */
    for (i = 8; i > 0; i--) 
    {
	if (name[i] == ' ') 
	    name[i] = '\0';
	else 
	    break;
    }

    /* remove the spaces from extensions */
    for (i = 3; i > 0; i--) 
    {
	if (extension[i] == ' ') 
	    extension[i] = '\0';
	else 
	    break;
    }

    if ((dirent->deAttributes & ATTR_WIN95LFN) == ATTR_WIN95LFN)
    {
	// ignore any long file name extension entries
	//
	// printf("Win95 long-filename entry seq 0x%0x\n", dirent->deName[0]);
    }
    else if ((dirent->deAttributes & ATTR_DIRECTORY) != 0) 
    {
        // don't deal with hidden directories; MacOS makes these
        // for trash directories and such; just ignore them.
	if ((dirent->deAttributes & ATTR_HIDDEN) != ATTR_HIDDEN)
        {
            strcpy(buffer, name);
            file_cluster = getushort(dirent->deStartCluster);
            followclust = file_cluster;
        }
    }
    else 
    {
        /*
         * a "regular" file entry
         * print attributes, size, starting cluster, etc.
         */
        strcpy(buffer, name);
        if (strlen(extension))  
        {
            strcat(buffer, ".");
            strcat(buffer, extension);
        }
    }

    return followclust;
}

/* write the values into a directory entry */
void write_dirent(struct direntry *dirent, char *filename, 
		  uint16_t start_cluster, uint32_t size)
{
    char *p, *p2;
    char *uppername;
    int len, i;

    /* clean out anything old that used to be here */
    memset(dirent, 0, sizeof(struct direntry));

    /* extract just the filename part */
    uppername = strdup(filename);
    p2 = uppername;
    for (i = 0; i < strlen(filename); i++) 
    {
	if (p2[i] == '/' || p2[i] == '\\') 
	{
	    uppername = p2+i+1;
	}
    }

    /* convert filename to upper case */
    for (i = 0; i < strlen(uppername); i++) 
    {
	uppername[i] = toupper(uppername[i]);
    }

    /* set the file name and extension */
    memset(dirent->deName, ' ', 8);
    p = strchr(uppername, '.');
    memcpy(dirent->deExtension, "___", 3);
    if (p == NULL) 
    {
	fprintf(stderr, "No filename extension given - defaulting to .___\n");
    }
    else 
    {
	*p = '\0';
	p++;
	len = strlen(p);
	if (len > 3) len = 3;
	memcpy(dirent->deExtension, p, len);
    }

    if (strlen(uppername)>8) 
    {
	uppername[8]='\0';
    }
    memcpy(dirent->deName, uppername, strlen(uppername));
    free(p2);

    /* set the attributes and file size */
    dirent->deAttributes = ATTR_NORMAL;
    putushort(dirent->deStartCluster, start_cluster);
    putulong(dirent->deFileSize, size);

    /* could also set time and date here if we really
       cared... */
}

/* create_dirent finds a free slot in the directory, and write the
   directory entry */
void create_dirent(struct direntry *dirent, char *filename, 
		   uint16_t start_cluster, uint32_t size,
		   uint8_t *image_buf, struct bpb33* bpb)
{
    while (1) 
    {
	if (dirent->deName[0] == SLOT_EMPTY) 
	{
	    /* we found an empty slot at the end of the directory */
	    write_dirent(dirent, filename, start_cluster, size);
	    dirent++;

	    /* make sure the next dirent is set to be empty, just in
	       case it wasn't before */
	    memset((uint8_t*)dirent, 0, sizeof(struct direntry));
	    dirent->deName[0] = SLOT_EMPTY;
	    return;
	}

	if (dirent->deName[0] == SLOT_DELETED) 
	{
	    /* we found a deleted entry - we can just overwrite it */
	    write_dirent(dirent, filename, start_cluster, size);
	    return;
	}
	dirent++;
    }
}
