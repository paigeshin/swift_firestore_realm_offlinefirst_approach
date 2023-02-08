# swift_firestore_realm_offlinefirst_approach

```swift
import Foundation
import FirebaseFirestore
import FirebaseFirestoreSwift
import RealmSwift
import Combine

class ItemObject: Object, ObjectKeyIdentifiable {
    @Persisted(primaryKey: true) var id: ObjectId
    @Persisted var title: String
    
    convenience init(title: String) {
        self.init()
        self.title = title
    }
    
    convenience init(id: ObjectId, title: String) {
        self.init()
        self.id = id
        self.title = title
    }
    
    func asNetworkModel() -> NetworkItem {
        NetworkItem(id: self.id.stringValue, title: self.title)
    }
    
    func asExternalModel() -> Item {
        Item(id: self.id.stringValue, title: self.title)
    }
}

struct NetworkItem: Codable {
    var id: String
    var title: String
    
    func asObject() throws -> ItemObject {
        ItemObject(id: try .init(string: self.id), title: self.title)
    }

    func asExternalModel() -> Item {
        Item(id: self.id, title: self.title)
    }
}

struct Item: Codable, Identifiable {
    
    var id: String
    var title: String
    
    init(id: String, title: String) {
        self.id = id
        self.title = title
    }
    
    func asObject() throws -> ItemObject {
        ItemObject(id: try .init(string: self.id), title: self.title)
    }
    
    func asNetworkModel() -> NetworkItem {
        NetworkItem(id: self.id, title: self.title)
    }
    
}

class ItemStore: ObservableObject {
    
    var subscriptions: Set<AnyCancellable> = []
    
    let realm = try! Realm()
    @Published var items: [Item] = []
    var listener: ListenerRegistration?
    
    func fetch() {
        self.listener = self.sync()
        self.realm
            .objects(ItemObject.self)
            .collectionPublisher
            .map({ realmResults in
                return realmResults.map { itemObject in
                    return itemObject.asExternalModel()
                }
            })
            .assertNoFailure()
            .receive(on: DispatchQueue.main)
            .assign(to: &$items)
    }
    
    func sync() -> ListenerRegistration {
        Firestore
            .firestore()
            .collection("items")
            .addSnapshotListener(includeMetadataChanges: true) { querySnapshot, error in
                
                guard let querySnapshot: QuerySnapshot = querySnapshot else {
                    return
                }
                
                for diff in querySnapshot.documentChanges {
                    switch diff.type {
                    case .added, .modified:
                        guard let networkItem = try? diff.document.data(as: NetworkItem.self) else {
                            continue
                        }
                        try? self.realm.write({
                            self.realm.add(try networkItem.asObject(), update: .modified)
                        })
                    case .removed:
                        guard let networkItem = try? diff.document.data(as: NetworkItem.self) else {
                            continue
                        }
                        try? self.realm.write({
                            guard let itemObject = self.realm.object(ofType: ItemObject.self, forPrimaryKey: try ObjectId(string: networkItem.id)) else {
                                return
                            }
                            self.realm.delete(itemObject)
                        })
                    }
                }

            }
    }
    
    func write(item: Item) {
        try? self.realm.write {
            self.realm.add(try item.asObject())
        }
    
        try? Firestore
            .firestore()
            .collection("items")
            .document(item.id)
            .setData(from: item.asNetworkModel(), completion: nil)
      
    }
    
    func update(item: Item) {
        try? self.realm.write({
            self.realm.add(try item.asObject(), update: .modified)
        })
 
        try? Firestore
            .firestore()
            .collection("items")
            .document(item.id)
            .setData(from: item.asNetworkModel(), merge: true, completion: nil)

    }
    
    func delete(item: Item) {
        try? self.realm.write({
            guard let itemObject = self.realm.object(ofType: ItemObject.self, forPrimaryKey: try ObjectId(string: item.id)) else {
                return
            }
            self.realm.delete(itemObject)
        })
        
        Firestore
            .firestore()
            .collection("items")
            .document(item.id)
            .delete(completion: nil)
    }
    
}


```
