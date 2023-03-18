# Upload image to firebase storage

###### Add dependencies
```
  // Library Android-Image-Cropper
  implementation 'com.theartofdev.edmodo:android-image-cropper:2.8.0'
  implementation 'com.squareup.okhttp:okhttp:2.7.5'
  implementation 'com.squareup.okhttp:okhttp-urlconnection:2.2.0'
  implementation 'com.squareup.picasso:picasso:2.5.2'
  
  // firebase storage
  implementation 'com.google.firebase:firebase-storage:19.1.1'
```

###### Add permission
```
  <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
  <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

###### Add Image-Cropper activity in manifest file
```
  <activity
    android:name="com.theartofdev.edmodo.cropper.CropImageActivity"
    android:theme="@style/Base.Theme.AppCompat" />
```

###### Add jcenter() in repositories Setting.gradle(project setting)

```
 repositories {
  jcenter()
}
```

###### global variables
```
  private Uri imageUri = null;
  private static String ImageURL = null;

  private FirebaseAuth firebaseAuth;
  private FirebaseFirestore firestore;
  private StorageReference storageReference;
  private String userID;
```

###### in onCreate
```
  firebaseAuth = FirebaseAuth.getInstance();
  firestore = FirebaseFirestore.getInstance();
  storageReference = FirebaseStorage.getInstance().getReference();
```

###### button call method 
```
  checkPermission();
```

###### to check if user allow permission or not 
```
 private void checkPermission() {

      //use permission to READ_EXTERNAL_STORAGE For Device >= Marshmallow
      if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {

          if (ContextCompat.checkSelfPermission(UpdateProfileActivity.this, Manifest.permission.READ_EXTERNAL_STORAGE) != PackageManager.PERMISSION_GRANTED) {
              Toast.makeText(UpdateProfileActivity.this, getString(R.string.permissionDenied), Toast.LENGTH_LONG).show();

              // to ask user to reade external storage
              ActivityCompat.requestPermissions(UpdateProfileActivity.this, new String[]{Manifest.permission.READ_EXTERNAL_STORAGE}, 1);

          } else {
              OpenGalleryImagePicker();
          }

          //implement code for device < Marshmallow
      } else {

          OpenGalleryImagePicker();
      }
  }
```

###### Method to open gallery
```
    private void OpenGalleryImagePicker() {
        // start picker to get image for cropping and then use the image in cropping activity
        Intent intent = CropImage.activity()
                .setGuidelines(CropImageView.Guidelines.ON)
                .getIntent(getActivity());
        someActivityResultLauncher.launch(intent);
    }
```

###### Activity result When user back Contains image
```
   // You can do the assignment inside onAttach or onCreate, i.e, before the activity is displayed
    ActivityResultLauncher<Intent> someActivityResultLauncher = registerForActivityResult(
            new ActivityResultContracts.StartActivityForResult(),
            new ActivityResultCallback<ActivityResult>() {
                @Override
                public void onActivityResult(ActivityResult result) {
                    if (result.getResultCode() == Activity.RESULT_OK) {
                        Intent data = result.getData();
                        CropImage.ActivityResult d = CropImage.getActivityResult(data);

                        imageUri = d.getUri();

                        // set image user in ImageView ;
                        profile_image.setImageURI(imageUri);

                        uploadImage();
                    }
                }
            });
```

###### upload image to StorageReference
```
    private void uploadImage() {

        if (firebaseAuth.getCurrentUser() != null) {

            // chick if user image is null or not
            if (imageUri != null) {

                userID = firebaseAuth.getCurrentUser().getUid();

                // mack progress bar dialog
                final ProgressDialog progressDialog = new ProgressDialog(this);
                progressDialog.setTitle(getString(R.string.uploading));
                progressDialog.setCancelable(false);
                progressDialog.show();

                // mack collection in fireStorage
                final StorageReference ref = storageReference.child("profile_image_user").child(userID + ".jpg");

                // get image user and give to imageUserPath
                ref.putFile(imageUri).addOnProgressListener(taskSnapshot -> {

                    double progress = (100.0 * taskSnapshot.getBytesTransferred()) / taskSnapshot.getTotalByteCount();
                    progressDialog.setMessage("upload " + (int) progress + "%");

                }).continueWithTask(task -> {
                    if (!task.isSuccessful()) {

                        throw task.getException();

                    }
                    return ref.getDownloadUrl();

                }).addOnCompleteListener(task -> {

                    if (task.isSuccessful()) {

                        progressDialog.dismiss();
                        Uri downloadUri = task.getResult();

                        assert downloadUri != null;
                        ImageURL = downloadUri.toString();
                        saveChange();

                    } else {

                        progressDialog.dismiss();
                        Toast.makeText(UpdateProfileActivity.this, " Error in addOnCompleteListener " + task.getException().getMessage(), Toast.LENGTH_SHORT).show();

                    }
                });
            }
        }
    }
```

###### update data in collection
```
    private void saveChange() {
        if (mAuth.getCurrentUser() != null) {
            progress_userImage.setVisibility(View.VISIBLE);
            userID = mAuth.getCurrentUser().getUid();

            final Map<String, Object> studentData = new HashMap<>();

            if (ImageURL != null && ImageURL.length() > 0) {
                studentData.put("imageUrl", ImageURL);
                profile_image.setImageURI(imageUri);
            }

            mCollection = db.collection("users").document(userID);
            mCollection.update(studentData)
                    .addOnCompleteListener(task -> {
                        if (task.isSuccessful()) {
                            progress_userImage.setVisibility(View.GONE);
                            Toast.makeText(getActivity(), "Update done", Toast.LENGTH_SHORT).show();
                        } else {
                            progress_userImage.setVisibility(View.GONE);
                            try {
                                Exception exception = task.getException();
                                assert exception != null;
                                throw exception;
                            } catch (FirebaseNetworkException net) {
                                Toast.makeText(getActivity(), "no Internet", Toast.LENGTH_SHORT).show();
                            } catch (Exception e) {
                                Toast.makeText(getActivity(), "Error " + e.getMessage(), Toast.LENGTH_SHORT).show();
                            }
                        }

                    });
        }
    }
```  
    
