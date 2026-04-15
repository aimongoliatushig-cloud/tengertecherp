# Mobile App Quick Start Guide

## 🚀 **Get Started in 1 Hour**

### **Step 1: Set Up Development Environment**

#### **Install Node.js & npm:**
```bash
# Download from https://nodejs.org/
# Install LTS version (18.x or newer)

# Verify installation
node --version
npm --version
```

#### **Install React Native CLI:**
```bash
npm install -g react-native-cli
```

#### **Install Android Studio (for Android development):**
- Download from https://developer.android.com/studio
- Install with Android SDK
- Set up Android Virtual Device (AVD)

#### **Install Xcode (for iOS development - macOS only):**
- From Mac App Store
- Requires macOS

### **Step 2: Create React Native Project**

```bash
# Create new project
npx react-native init MunicipalMobileApp --template react-native-template-typescript

# Navigate to project
cd MunicipalMobileApp

# Install dependencies
npm install
```

### **Step 3: Install Required Libraries**

```bash
# Navigation
npm install @react-navigation/native @react-navigation/stack
npm install react-native-screens react-native-safe-area-context

# UI Components
npm install react-native-elements
npm install react-native-vector-icons

# HTTP Client
npm install axios

# Notifications
npm install @react-native-firebase/app
npm install @react-native-firebase/messaging

# Camera & Media
npm install react-native-image-picker
npm install react-native-audio-recorder-player

# Storage
npm install @react-native-async-storage/async-storage

# Location
npm install react-native-geolocation-service

# Link libraries (for older React Native versions)
npx react-native link
```

### **Step 4: Configure Firebase for Push Notifications**

1. **Create Firebase Project:**
   - Go to https://console.firebase.google.com/
   - Create new project "Municipal Operations"
   - Add Android app (get package name from AndroidManifest.xml)
   - Add iOS app (get bundle ID from Xcode)

2. **Download Configuration Files:**
   - `google-services.json` for Android
   - `GoogleService-Info.plist` for iOS

3. **Place files in project:**
   ```
   MunicipalMobileApp/
   ├── android/app/google-services.json
   └── ios/GoogleService-Info.plist
   ```

### **Step 5: Connect to Odoo API**

Create `src/api/odoo.ts`:
```typescript
import axios from 'axios';

const ODOO_URL = 'http://localhost:8069';
const ODOO_DB = 'odoo19';

export class OdooAPI {
  private uid: number | null = null;
  private sessionId: string | null = null;

  async login(username: string, password: string): Promise<boolean> {
    try {
      const response = await axios.post(`${ODOO_URL}/web/session/authenticate`, {
        jsonrpc: '2.0',
        params: {
          db: ODOO_DB,
          login: username,
          password: password,
        },
      });
      
      this.uid = response.data.result.uid;
      this.sessionId = response.data.result.session_id;
      return true;
    } catch (error) {
      console.error('Login failed:', error);
      return false;
    }
  }

  async getTasks(): Promise<any[]> {
    if (!this.uid) throw new Error('Not authenticated');
    
    const response = await axios.post(`${ODOO_URL}/jsonrpc`, {
      jsonrpc: '2.0',
      method: 'call',
      params: {
        service: 'object',
        method: 'execute_kw',
        args: [
          ODOO_DB,
          this.uid,
          'password',
          'project.task',
          'search_read',
          [[]],
          {
            fields: ['id', 'name', 'description', 'progress_percentage', 'stage_id'],
          },
        ],
      },
    });
    
    return response.data.result;
  }

  async submitReport(taskId: number, data: any): Promise<boolean> {
    // Implement report submission
    return true;
  }
}

export const odooAPI = new OdooAPI();
```

### **Step 6: Create Basic Screens**

#### **Login Screen (`src/screens/Login.tsx`):**
```typescript
import React, { useState } from 'react';
import { View, TextInput, Button, Text, StyleSheet } from 'react-native';
import { odooAPI } from '../api/odoo';

export const LoginScreen = ({ navigation }: any) => {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  const handleLogin = async () => {
    setLoading(true);
    setError('');
    
    try {
      const success = await odooAPI.login(username, password);
      if (success) {
        navigation.navigate('Dashboard');
      } else {
        setError('Нэвтрэх нэр эсвэл нууц үг буруу');
      }
    } catch (err) {
      setError('Серверт холбогдоход алдаа гарлаа');
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Municipal Operations</Text>
      <TextInput
        style={styles.input}
        placeholder="Нэвтрэх нэр"
        value={username}
        onChangeText={setUsername}
      />
      <TextInput
        style={styles.input}
        placeholder="Нууц үг"
        secureTextEntry
        value={password}
        onChangeText={setPassword}
      />
      {error ? <Text style={styles.error}>{error}</Text> : null}
      <Button
        title={loading ? 'Нэвтэрч байна...' : 'Нэвтрэх'}
        onPress={handleLogin}
        disabled={loading}
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    padding: 20,
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    textAlign: 'center',
    marginBottom: 30,
  },
  input: {
    borderWidth: 1,
    borderColor: '#ccc',
    padding: 10,
    marginBottom: 15,
    borderRadius: 5,
  },
  error: {
    color: 'red',
    marginBottom: 15,
    textAlign: 'center',
  },
});
```

#### **Dashboard Screen (`src/screens/Dashboard.tsx`):**
```typescript
import React, { useEffect, useState } from 'react';
import { View, Text, FlatList, StyleSheet } from 'react-native';
import { odooAPI } from '../api/odoo';

export const DashboardScreen = () => {
  const [tasks, setTasks] = useState<any[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadTasks();
  }, []);

  const loadTasks = async () => {
    try {
      const taskList = await odooAPI.getTasks();
      setTasks(taskList);
    } catch (error) {
      console.error('Failed to load tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const renderTask = ({ item }: { item: any }) => (
    <View style={styles.taskCard}>
      <Text style={styles.taskName}>{item.name}</Text>
      <Text>Явц: {item.progress_percentage}%</Text>
    </View>
  );

  if (loading) {
    return (
      <View style={styles.container}>
        <Text>Ачааллаж байна...</Text>
      </View>
    );
  }

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Миний даалгаварууд</Text>
      <FlatList
        data={tasks}
        renderItem={renderTask}
        keyExtractor={(item) => item.id.toString()}
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 20,
  },
  title: {
    fontSize: 20,
    fontWeight: 'bold',
    marginBottom: 20,
  },
  taskCard: {
    backgroundColor: 'white',
    padding: 15,
    marginBottom: 10,
    borderRadius: 5,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,
  },
  taskName: {
    fontSize: 16,
    fontWeight: 'bold',
    marginBottom: 5,
  },
});
```

### **Step 7: Set Up Navigation**

Create `src/navigation/AppNavigator.tsx`:
```typescript
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';
import { LoginScreen } from '../screens/Login';
import { DashboardScreen } from '../screens/Dashboard';

const Stack = createStackNavigator();

export const AppNavigator = () => {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Login">
        <Stack.Screen 
          name="Login" 
          component={LoginScreen}
          options={{ headerShown: false }}
        />
        <Stack.Screen 
          name="Dashboard" 
          component={DashboardScreen}
          options={{ title: 'Хянах самбар' }}
        />
      </Stack.Navigator>
    </NavigationContainer>
  );
};
```

### **Step 8: Update App.tsx**

```typescript
import React from 'react';
import { AppNavigator } from './src/navigation/AppNavigator';

const App = () => {
  return <AppNavigator />;
};

export default App;
```

### **Step 9: Run the App**

#### **Android:**
```bash
# Start Metro bundler
npx react-native start

# In another terminal, run Android
npx react-native run-android
```

#### **iOS (macOS only):**
```bash
cd ios
pod install
cd ..
npx react-native run-ios
```

### **Step 10: Test with Odoo**

1. **Start Odoo:** `docker-compose up -d` in odoo19-project
2. **Create test user** in Odoo
3. **Create test tasks** in Odoo
4. **Login** in mobile app with test credentials
5. **Verify tasks** appear in dashboard

## 🎯 **Next Features to Add:**

### **Week 1:**
1. **Push notifications** setup
2. **Task details** screen
3. **Image upload** for reports

### **Week 2:**
1. **Audio recording** for field reports
2. **Offline mode** with SQLite
3. **Notification preferences**

### **Week 3:**
1. **GPS location** tracking
2. **Emergency job** handling
3. **Real-time updates**

## 🔧 **Troubleshooting:**

### **Common Issues:**

1. **Android build fails:**
   ```bash
   cd android
   ./gradlew clean
   cd ..
   npx react-native run-android
   ```

2. **iOS build fails:**
   ```bash
   cd ios
   pod deintegrate
   pod install
   cd ..
   ```

3. **Metro bundler issues:**
   ```bash
   watchman watch-del-all
   rm -rf node_modules
   npm install
   npx react-native start --reset-cache
   ```

4. **Odoo connection issues:**
   - Check Odoo is running: http://localhost:8069
   - Verify database name in API config
   - Check firewall/port settings

## 📱 **Testing on Real Devices:**

### **Android:**
1. Enable USB debugging on phone
2. Connect via USB
3. Run: `npx react-native run-android`

### **iOS:**
1. Connect iPhone via USB
2. Open Xcode project
3. Select your device
4. Build and run

## 🚀 **Production Deployment:**

### **Android Play Store:**
1. Generate signed APK
2. Create Play Store listing
3. Submit for review
4. Publish

### **iOS App Store:**
1. Create App Store Connect listing
2. Archive build in Xcode
3. Submit for review
4. Publish

## 📞 **Need Help?**

### **Resources:**
- React Native Docs: https://reactnative.dev/docs/getting-started
- Odoo API Docs: https://www.odoo.com/documentation/19.0/developer/misc/api/odoo.html
- Firebase Docs: https://firebase.google.com/docs

### **Community:**
- React Native Community: https://github.com/react-native-community
- Stack Overflow: Use tags [react-native], [odoo]
- Discord: React Native community servers

## 🎉 **You're Ready to Build!**

The foundation is set. You now have:
1. ✅ Development environment
2. ✅ React Native project
3. ✅ Odoo API connection
4. ✅ Basic screens
5. ✅ Navigation setup

**Start coding and transform your municipal operations with mobile technology!**