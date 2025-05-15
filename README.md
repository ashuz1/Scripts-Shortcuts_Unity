# Scripts-Shortcuts_Unity


XXXXXXXX__________________________________________________________________________________________________________________________________Editor Shortcut
using System.Collections.Generic;

using UnityEditor;
using UnityEngine;
using System.IO;
using System.Collections.Generic;

[InitializeOnLoad]
public static class EditorShortcuts
{
    private static List<GameObject> selectionHistory = new List<GameObject>();
    private const int MAX_HISTORY = 100;

    static EditorShortcuts()
    {
        // Initialize selection history tracking
        Selection.selectionChanged += () =>
        {
            if (Selection.activeGameObject != null)
            {
                selectionHistory.Add(Selection.activeGameObject);
                if (selectionHistory.Count > MAX_HISTORY)
                {
                    selectionHistory.RemoveAt(0);
                }
            }
        };
    }

    // Toggle current selection active/inactive
    [MenuItem("Custom Tools/Toggle Active State &z")] // Alt+Z
    private static void ToggleActiveState()
    {
        foreach (GameObject obj in Selection.gameObjects)
        {
            Undo.RecordObject(obj, "Toggle Active State");
            obj.SetActive(!obj.activeSelf);
        }
    }

    // Create a new folder in the selected path or in Assets
    [MenuItem("Custom Tools/Create New Folder %#n")] // Ctrl+Shift+N
    private static void CreateNewFolder()
    {
        string folderPath = "Assets";

        if (Selection.activeObject != null)
        {
            string path = AssetDatabase.GetAssetPath(Selection.activeObject);
            if (!string.IsNullOrEmpty(path) && AssetDatabase.IsValidFolder(path))
            {
                folderPath = path;
            }
            else
            {
                string maybeFolder = Path.GetDirectoryName(path);
                if (AssetDatabase.IsValidFolder(maybeFolder))
                {
                    folderPath = maybeFolder;
                }
            }
        }

        string newFolderPath = AssetDatabase.GenerateUniqueAssetPath(Path.Combine(folderPath, "New Folder"));
        AssetDatabase.CreateFolder(folderPath, Path.GetFileName(newFolderPath));
        AssetDatabase.Refresh();

        Debug.Log($"Created folder: {newFolderPath}");
    }

    [MenuItem("Custom Tools/Toggle Inspector Lock &x")] // Alt+X
    private static void ToggleInspectorLock()
    {
        var tracker = ActiveEditorTracker.sharedTracker;
        if (tracker != null)
        {
            tracker.isLocked = !tracker.isLocked;
            tracker.ForceRebuild();

            // Refresh all inspector windows
            var inspectorType = typeof(UnityEditor.Editor).Assembly.GetType("UnityEditor.InspectorWindow");
            var inspectors = Resources.FindObjectsOfTypeAll(inspectorType);
            foreach (EditorWindow inspector in inspectors)
            {
                inspector.Repaint();
            }
        }
    }

    // Go to last selected object in hierarchy
    [MenuItem("Custom Tools/Go To Last Selected &s")] // Alt+s
    private static void GoToLastSelected()
    {
        if (selectionHistory.Count > 1)
        {
            GameObject lastSelected = selectionHistory[selectionHistory.Count - 2];
            if (lastSelected != null)
            {
                Selection.activeGameObject = lastSelected;
                EditorGUIUtility.PingObject(lastSelected);
            }
        }
    }
}
XXXXXXXX__________________________________________________________________________________________________________________________________Firerbase firestore
using System.Collections.Generic;
using UnityEngine;
using Firebase;
using Firebase.Firestore;
using Firebase.Extensions;
using System.Threading.Tasks;

[RequireComponent(typeof(Firebase.FirebaseApp))]
public class FirebaseTest : MonoBehaviour
{
    FirebaseFirestore db;

    async void Start()
    {
        await InitializeFirebase();
        await FetchCollectionWithKnownSubcollections("Users");
        FetchAndPrintCollectionFromPath("Users/U1/tt1/q1");
    }

    async Task InitializeFirebase()
    {
        var dependencyStatus = await FirebaseApp.CheckAndFixDependenciesAsync();
        if (dependencyStatus == DependencyStatus.Available)
        {
            db = FirebaseFirestore.DefaultInstance;
            Debug.Log("✅ Firebase initialized.");
        }
        else
        {
            Debug.LogError($"❌ Firebase dependency error: {dependencyStatus}");
        }
    }

    async Task FetchCollectionWithKnownSubcollections(string collectionName)
    {
        CollectionReference mainCollection = db.Collection(collectionName);
        QuerySnapshot snapshot = await mainCollection.GetSnapshotAsync();

        if (snapshot.Count == 0)
        {
            Debug.LogWarning($"⚠️ No documents found in collection '{collectionName}'.");
            return;
        }

        foreach (DocumentSnapshot document in snapshot.Documents)
        {
            Debug.Log($"📄 Document ID: {document.Id}");
            Dictionary<string, object> docData = document.ToDictionary();

            foreach (var pair in docData)
            {
                Debug.Log($"   🔹 {pair.Key}: {pair.Value}");
            }

            if (document.ContainsField("subcollections"))
            {
                List<string> subcollectionNames = document.GetValue<List<string>>("subcollections");
                await FetchSubcollections(document.Reference, subcollectionNames);
            }
            else
            {
                Debug.Log($"⚠️ Document '{document.Id}' has no 'subcollections' field.");
            }
        }
    }

    async Task FetchSubcollections(DocumentReference docRef, List<string> subcollectionNames)
    {
        foreach (string subName in subcollectionNames)
        {
            CollectionReference subCol = docRef.Collection(subName);
            QuerySnapshot subSnap = await subCol.GetSnapshotAsync();

            if (subSnap.Count == 0)
            {
                Debug.Log($"   ⚠️ Subcollection '{subName}' is empty.");
                continue;
            }

            foreach (DocumentSnapshot subDoc in subSnap.Documents)
            {
                Debug.Log($"   📂 Subcollection '{subName}', Doc ID: {subDoc.Id}");
                Dictionary<string, object> subDocData = subDoc.ToDictionary();

                foreach (var pair in subDocData)
                {
                    Debug.Log($"      🔸 {pair.Key}: {pair.Value}");
                }
            }
        }
    }
    public async void FetchAndPrintCollectionFromPath(string path)
    {
        if (db == null)
        {
            Debug.LogError("❌ Firebase not initialized.");
            return;
        }

        Debug.Log($"📥 Fetching from path: '{path}'");

        // Split the path into segments
        string[] segments = path.Split('/');

        // Validate that the path is correct (odd = document, even = collection)
        if (segments.Length % 2 != 0)
        {
            Debug.LogError("❌ Invalid path. Must end with a collection.");
            return;
        }

        // Traverse to the collection
        CollectionReference colRef = null;
        DocumentReference docRef = null;

        for (int i = 0; i < segments.Length; i++)
        {
            if (i % 2 == 0)
            {
                // Collection
                if (docRef == null)
                    colRef = db.Collection(segments[i]);
                else
                    colRef = docRef.Collection(segments[i]);
            }
            else
            {
                // Document
                docRef = colRef.Document(segments[i]);
            }
        }

        // Now colRef is pointing to the final collection
        QuerySnapshot snapshot = await colRef.GetSnapshotAsync();

        if (snapshot.Count == 0)
        {
            Debug.LogWarning($"⚠️ No documents found at '{path}'.");
            return;
        }

        foreach (DocumentSnapshot document in snapshot.Documents)
        {
            Debug.Log($"📄 Document ID: {document.Id}");
            Dictionary<string, object> docData = document.ToDictionary();

            foreach (var pair in docData)
            {
                Debug.Log($"   🔹 {pair.Key}: {pair.Value}");
            }

            if (document.ContainsField("subcollections"))
            {
                List<string> subcollectionNames = document.GetValue<List<string>>("subcollections");
                await FetchSubcollections(document.Reference, subcollectionNames);
            }
        }
    }

}


XXXXXXXX__________________________________________________________________________________________________________________________________Firerbase LeaderBoard

using Firebase;
using Firebase.Extensions;
using Firebase.Firestore;
using System.Collections.Generic;
using TMPro;
using UnityEngine;
using UnityEngine.UI;
using System;


public class FireBaseLeaderBoard : MonoBehaviour
{

    FirebaseFirestore db;

    [SerializeField] TMP_Text RankTxt;
    [SerializeField] TMP_InputField NameInput;
    [SerializeField] Button SaveBtn;
    [SerializeField] GameObject LeaderboardCard;
    [SerializeField] Transform cardParent;
    private LeaderboardCard lastCard;
    // This function will be called on start to initialize Firebase.

    private int lastRank, lastScore;
    void OnEnable()
    {
        FirebaseApp.CheckAndFixDependenciesAsync().ContinueWithOnMainThread(task =>
        {
            if (task.Result == DependencyStatus.Available)
            {
                FirebaseApp app = FirebaseApp.DefaultInstance;
                db = FirebaseFirestore.DefaultInstance; // ✅ FIX: assign db properly here
                Debug.Log("Firebase Initialized!");

                if (!string.IsNullOrEmpty(PlayerPrefs.GetString("UserName")))
                {
                    NameInput.text = PlayerPrefs.GetString("UserName");
                    _setData(); // ✅ Only safe after db is assigned
                }

                LoadTopUsers(); // ✅ Also safe after db is assigned
            }
            else
            {
                Debug.LogError("Could not resolve Firebase dependencies: " + task.Result);
            }
        });
    }
    
    public void LoadTopUsers(int leaderboardSize = 120)
    {
        db.Collection("users")
          .OrderByDescending("score")
          .Limit(leaderboardSize)
          .GetSnapshotAsync()
          .ContinueWithOnMainThread(task =>
          {
              if (task.IsFaulted)
              {
                  Debug.LogError("Error fetching leaderboard: " + task.Exception);
                 // ShowErrorUI("Failed to load leaderboard. Please try again.");
                  return;
              }

          // Clear old cards
          foreach (Transform child in cardParent)
              {
                  Destroy(child.gameObject);
              }

              int rank = 1;
              LeaderboardCard lastCard = null;

              foreach (DocumentSnapshot doc in task.Result.Documents)
              {
                  var data = doc.ToDictionary();
                  if (!data.TryGetValue("name", out object nameObj) ||
                      !data.TryGetValue("score", out object scoreObj))
                  {
                      Debug.LogWarning($"Skipping invalid user data in doc {doc.Id}");
                      continue;
                  }

                  string name = nameObj.ToString();
                  if (!int.TryParse(scoreObj.ToString(), out int score))
                  {
                      Debug.LogWarning($"Invalid score for user {name}");
                      continue;
                  }

                  //     Debug.Log($"Rank #{rank}: {name} - {score}");

                  if (lastScore != score)
                  {
                      lastRank = rank;
                      lastScore = score;
                  }
                  if (rank % 2 == 1 || lastCard == null)
                  {
                      GameObject obj = Instantiate(LeaderboardCard, cardParent);
                      lastCard = obj.GetComponent<LeaderboardCard>();
                      lastCard.SetDataCardOne(lastRank, name, score.ToString());
                  }
                  else
                  {
                      lastCard.SetDataCardTwo(lastRank, name, score.ToString(), true);
                  }

                  rank++;
                  
              }
          });
    }

    public void _setData()
    {
        if (!string.IsNullOrEmpty(NameInput.text))
        {
            PlayerPrefs.SetString("UserName", NameInput.text);

            string deviceID = GetDeviceID();

            DocumentReference docRef = db.Collection("users").Document(deviceID);

            Dictionary<string, object> user = new Dictionary<string, object>
            {
                { "name", NameInput.text },
                { "score", PlayerPrefs.GetInt("TotalScore") }
            };

            docRef.SetAsync(user).ContinueWithOnMainThread(task =>
            {
                if (task.IsCompleted && !task.IsFaulted)
                {
                    Debug.Log("User data saved/updated in Firestore!");
                }
                else
                {
                    Debug.LogError("Failed to write to Firestore: " + task.Exception);
                }
            });

            docRef.GetSnapshotAsync().ContinueWithOnMainThread(task =>
            {
                if (task.IsCompleted && task.Result.Exists)
                {
                    Dictionary<string, object> data = task.Result.ToDictionary();
                    Debug.Log("Fetched from Firestore:");
                    Debug.Log("User name: " + data["name"]);
                    Debug.Log("Score: " + data["score"]);
                   
                }
            });
        }
        GetCurrentUserRank(GetDeviceID());
    }

    private string GetDeviceID()
    {
        if (PlayerPrefs.HasKey("DeviceID"))
        {
            return PlayerPrefs.GetString("DeviceID");
        }
        else
        {
            string newID = Guid.NewGuid().ToString();
            PlayerPrefs.SetString("DeviceID", newID);
            return newID;
        }
    }   
    void GetCurrentUserRank(string userId)
    {
        DocumentReference docRef = db.Collection("users").Document(userId);
        docRef.GetSnapshotAsync().ContinueWithOnMainThread(task =>
        {
            if (!task.Result.Exists)
            {
                Debug.LogWarning("User not found.");
                return;
            }

            var userData = task.Result.ToDictionary();
            int currentUserScore = int.Parse(userData["score"].ToString());

            // Count how many users have a higher score
            db.Collection("users")
              .WhereGreaterThan("score", currentUserScore)
              .GetSnapshotAsync()
              .ContinueWithOnMainThread(rankTask =>
              {
                  if (rankTask.IsFaulted)
                  {
                      Debug.LogError("Error fetching user rank: " + rankTask.Exception);
                      return;
                  }

                  int usersAbove = rankTask.Result.Count;
                  int currentUserRank = usersAbove + 1;
                  RankTxt.text = currentUserRank.ToString();
                 // Debug.Log($"Current user rank: #{currentUserRank}");
              });
        });
    }

}



