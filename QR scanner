import React, { useState, useEffect, useCallback } from 'react';
import {
  View,
  Text,
  StyleSheet,
  TouchableOpacity,
  Modal,
  FlatList,
  Alert,
  Linking,
  SafeAreaView,
  StatusBar,
  ScrollView,
  ImageBackground,
} from 'react-native';
import { CameraView, Camera } from 'expo-camera';
import AsyncStorage from '@react-native-async-storage/async-storage';
import Animated, {
  useSharedValue,
  withSequence,
  withTiming,
  withRepeat,
  useAnimatedStyle,
  withDelay,
  interpolateColor,
  Easing,
} from 'react-native-reanimated';
import { initializeApp } from 'firebase/app';
import {
  getFirestore,
  doc,
  onSnapshot,
  increment,
  updateDoc,
  setDoc,
  collection,
  addDoc,
  serverTimestamp,
} from 'firebase/firestore';
import { getAnalytics, logEvent } from 'firebase/analytics';
import * as Crashlytics from 'expo-firebase-crashlytics';

const brandPink = '#FF4D8D';
const brandPinkDark = '#D6336C';
const brandPinkLight = '#FFB3CB';

const firebaseConfig = {
  apiKey: 'AIzaSyCobXxBiL4FQAzm7zQaKnRYEhoMF1q3PQE',
  authDomain: 'ur-scanner-b3420.firebaseapp.com',
  projectId: 'ur-scanner-b3420',
  storageBucket: 'ur-scanner-b3420.firebasestorage.app',
  messagingSenderId: '12289506045',
  appId: '1:12289506045:web:626749a9595ca9349adc94',
  measurementId: 'G-625FPTRVRF',
};
const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const analytics = getAnalytics(app);

const WALLETS_RAW = [
  { name: 'BDO', deeplink: 'bdo://', store: 'com.bdo.unibank', color: '#0033A0' },
  { name: 'BPI', deeplink: 'bpi://', store: 'com.bpi.mobile', color: '#A12830' },
  { name: 'Chinabank', deeplink: 'chinabank://', store: 'com.chinabank.mobile', color: '#E31837' },
  { name: 'GCash', deeplink: 'gcash://qrcode/scan', store: 'com.globe.gcash.android', color: '#007DFE' },
  { name: 'GrabPay', deeplink: 'grab://', store: 'com.grabtaxi.passenger', color: '#00B14F' },
  { name: 'Landbank', deeplink: 'landbank://', store: 'com.landbank.mobile', color: '#006937' },
  { name: 'Maya', deeplink: 'maya://qr', store: 'com.paymaya', color: '#00D631' },
  { name: 'Metrobank', deeplink: 'metrobank://', store: 'com.metrobank.mobile', color: '#002A5C' },
  { name: 'PNB', deeplink: 'pnb://', store: 'com.pnb.mobile', color: '#0066B3' },
  { name: 'RCBC', deeplink: 'rcbc://', store: 'com.rcbc.mobile', color: '#003087' },
  { name: 'Security Bank', deeplink: 'securitybank://', store: 'com.securitybank.mobile', color: '#009CDE' },
  { name: 'ShopeePay', deeplink: 'shopee://', store: 'com.shopee.ph', color: '#EE4D2D' },
  { name: 'UnionBank', deeplink: 'unionbank://', store: 'com.unionbank.ph', color: '#FF7F00' },
];
const WALLETS_SORTED = [...WALLETS_RAW].sort((a, b) => a.name.localeCompare(b.name));

const FIREWALL = {
  MIN_INTERVAL_MS: 5000,
  MAX_DAILY_CLICKS: 100,
  DAILY_KEY: 'daily_clicks',
  DAILY_DATE_KEY: 'daily_clicks_date',
  LAST_CLICK_KEY: 'lastClickTime',
};

async function firewallAllow() {
  const now = Date.now();

  const lastClick = await AsyncStorage.getItem(FIREWALL.LAST_CLICK_KEY);
  if (lastClick && now - parseInt(lastClick, 10) < FIREWALL.MIN_INTERVAL_MS) {
    Alert.alert('Dahan-dahan lang', 'Maghintay ng 5 segundo bago ulit mag-select.');
    return false;
  }

  const today = new Date().toISOString().slice(0, 10);
  const savedDate = await AsyncStorage.getItem(FIREWALL.DAILY_DATE_KEY);
  let dailyClicks = parseInt(await AsyncStorage.getItem(FIREWALL.DAILY_KEY) || '0', 10);

  if (savedDate !== today) {
    dailyClicks = 0;
    await AsyncStorage.setItem(FIREWALL.DAILY_DATE_KEY, today);
  }

  if (dailyClicks >= FIREWALL.MAX_DAILY_CLICKS) {
    Alert.alert('Daily limit reached', 'Bumalik bukas.');
    return false;
  }

  await AsyncStorage.multiSet([
    [FIREWALL.LAST_CLICK_KEY, now.toString()],
    [FIREWALL.DAILY_KEY, (dailyClicks + 1).toString()],
  ]);
  return true;
}

function getEMVTag(data, tag) {
  let i = 0;
  while (i < data.length - 4) {
    const id = data.substr(i, 2);
    const len = parseInt(data.substr(i + 2, 2), 10);
    if (isNaN(len)) break;
    const value = data.substr(i + 4, len);
    if (id === tag) return value;
    i += 4 + len;
  }
  return null;
}

function extractMerchantName(qrData) {
  try {
    const name = getEMVTag(qrData, '59');
    return name && name.trim() ? name.trim() : 'Unknown Merchant';
  } catch (e) {
    Crashlytics.recordError(e);
    return 'Unknown Merchant';
  }
}

function crc16ccitt(data) {
  let crc = 0xffff;
  for (let i = 0; i < data.length; i++) {
    crc ^= data.charCodeAt(i) << 8;
    for (let b = 0; b < 8; b++) {
      crc = (crc & 0x8000) ? ((crc << 1) ^ 0x1021) : (crc << 1);
      crc &= 0xffff;
    }
  }
  return crc.toString(16).toUpperCase().padStart(4, '0');
}

function isValidQRPH(data) {
  if (!data || data.length > 512 || data.length < 10) return false;
  try {
    if (!data.startsWith('000201')) return false;
    if (getEMVTag(data, '63') === null) return false;
    const payloadWithoutCrc = data.slice(0, -4);
    const declaredCrc = data.slice(-4).toUpperCase();
    const computedCrc = crc16ccitt(payloadWithoutCrc + '6304');
    return declaredCrc === computedCrc;
  } catch {
    return data.startsWith('000201') && data.includes('63');
  }
}

const DEFAULT_STATS = WALLETS_RAW.reduce(
  (acc, w) => {
    acc['selected_' + w.name.replace(/\s/g, '_')] = 0;
    return acc;
  },
  { total_app_select_clicks: 0, selected_QRPH_Default: 0 }
);

export default function App() {
  const [hasPermission, setHasPermission] = useState(null);
  const [scanned, setScanned] = useState(false);
  const [showSettings, setShowSettings] = useState(false);
  const [showLiveStats, setShowLiveStats] = useState(false);
  const [showAbout, setShowAbout] = useState(false);
  const [showPrivacy, setShowPrivacy] = useState(false);
  const [showQRList, setShowQRList] = useState(false);
  const [showWalletSelect, setShowWalletSelect] = useState(false);
  const [currentQR, setCurrentQR] = useState(null);
  const [savedQRs, setSavedQRs] = useState([]);
  const [liveStats, setLiveStats] = useState(DEFAULT_STATS);

  useEffect(() => {
    (async () => {
      const { status } = await Camera.requestCameraPermissionsAsync();
      setHasPermission(status === 'granted');
      loadSavedQRs();
      Crashlytics.setUserId('anonymous_user');
      Crashlytics.log('UR Scanner launched');
      logEvent(analytics, 'app_open');
    })();
  }, []);

  useEffect(() => {
    const ref = doc(db, 'live_stats', 'summary');
    const unsub = onSnapshot(
      ref,
      (snap) => {
        if (snap.exists()) setLiveStats(snap.data());
        else setDoc(ref, DEFAULT_STATS);
      },
      (err) => Crashlytics.recordError(err)
    );
    return () => unsub();
  }, []);

  const loadSavedQRs = async () => {
    try {
      const raw = await AsyncStorage.getItem('qr_list');
      if (raw) setSavedQRs(JSON.parse(raw));
    } catch (e) {
      Crashlytics.recordError(e);
    }
  };

  const saveQRToList = async (qrData) => {
    try {
      const raw = await AsyncStorage.getItem('qr_list');
      let list = raw ? JSON.parse(raw) : [];
      if (list.some((i) => i.data === qrData)) return;
      list.unshift({
        name: extractMerchantName(qrData),
        data: qrData,
        date: new Date().toISOString(),
      });
      await AsyncStorage.setItem('qr_list', JSON.stringify(list));
      setSavedQRs(list);
    } catch (e) {
      Crashlytics.recordError(e);
    }
  };

  const handleBarCodeScanned = useCallback(
    async ({ data }) => {
      if (scanned) return;
      if (!isValidQRPH(data)) {
        Alert.alert('Invalid QR', 'Hindi ito valid QRPH code. Baka scam.');
        Crashlytics.recordError(new Error('Invalid QRPH scanned'));
        return;
      }
      setScanned(true);
      setCurrentQR(data);
      saveQRToList(data);
      setShowWalletSelect(true);
      Crashlytics.log('QR Scanned: ' + extractMerchantName(data));
      logEvent(analytics, 'qr_scanned');
    },
    [scanned]
  );

  const handleWalletSelect = async (walletName) => {
    const allowed = await firewallAllow();
    if (!allowed) return;

    setShowWalletSelect(false);

    try {
      Crashlytics.log('User selected: ' + walletName);
      logEvent(analytics, 'wallet_select', { wallet: walletName });

      const field = 'selected_' + walletName.replace(/\s/g, '_');
      try {
        await updateDoc(doc(db, 'live_stats', 'summary'), {
          total_app_select_clicks: increment(1),
          [field]: increment(1),
        });
      } catch (err) {
        const raw = await AsyncStorage.getItem('pending_clicks');
        const pending = raw ? JSON.parse(raw) : [];
        pending.push({ wallet: walletName, timestamp: Date.now() });
        await AsyncStorage.setItem('pending_clicks', JSON.stringify(pending));
        Crashlytics.recordError(err);
      }

      await addDoc(collection(db, 'click_logs'), {
        action: 'clicked_open_in_' + walletName,
        merchant: extractMerchantName(currentQR),
        qr_hash: currentQR ? currentQR.slice(-6) : '',
        timestamp: serverTimestamp(),
        disclaimer: 'User clicked button only. No payment confirmed.',
      });

      const url =
        walletName === 'QRPH_Default'
          ? 'https://qrph.co?data=' + encodeURIComponent(currentQR)
          : WALLETS_RAW.find((w) => w.name === walletName)?.deeplink;

      if (!url) throw new Error('No deeplink for ' + walletName);

      const canOpen = await Linking.canOpenURL(url);
      if (canOpen) {
        await Linking.openURL(url);
      } else {
        const wallet = WALLETS_RAW.find((w) => w.name === walletName);
        Alert.alert(
          walletName + ' not installed',
          'I-install mula sa Play Store?',
          [
            { text: 'Cancel', style: 'cancel' },
            {
              text: 'Install',
              onPress: () =>
                Linking.openURL('market://details?id=' + (wallet?.store || '')),
            },
          ]
        );
      }
    } catch (err) {
      Crashlytics.recordError(err);
      Alert.alert('Error', 'Hindi ma-open ang app. Subukan ulit.');
    }

    setTimeout(() => setScanned(false), 2000);
  };

  const deleteQR = async (index) => {
    try {
      const list = [...savedQRs];
      list.splice(index, 1);
      await AsyncStorage.setItem('qr_list', JSON.stringify(list));
      setSavedQRs(list);
    } catch (e) {
      Crashlytics.recordError(e);
    }
  };

  const openQR = (qrData) => {
    setShowQRList(false);
    setCurrentQR(qrData);
    setShowWalletSelect(true);
  };

  if (hasPermission === null) {
    return (
      <View style={styles.container}>
        <Text style={styles.permText}>Requesting camera permission...</Text>
      </View>
    );
  }
  if (hasPermission === false) {
    return (
      <View style={styles.container}>
        <Text style={styles.permText}>Camera access denied.</Text>
      </View>
    );
  }

  const wavyStats = [
    { label: 'Total App Select Clicks', value: liveStats.total_app_select_clicks, color: '#FFD600' },
    ...WALLETS_SORTED.map((w) => ({
      label: 'Selected ' + w.name,
      value: liveStats['selected_' + w.name.replace(/\s/g, '_')] || 0,
      color: w.color,
    })),
    { label: 'Selected Other QRPH Apps', value: liveStats.selected_QRPH_Default || 0, color: '#888' },
  ];

  return (
    <SafeAreaView style={styles.container}>
      <StatusBar barStyle="light-content" />

      <CameraView
        style={StyleSheet.absoluteFillObject}
        onBarcodeScanned={scanned ? undefined : handleBarCodeScanned}
        barcodeScannerSettings={{ barcodeTypes: ['qr'] }}
      />

      <TouchableOpacity
        style={styles.settingsBtn}
        onPress={() => setShowSettings(true)}
      >
        <Text style={styles.iconText}>⚙️</Text>
      </TouchableOpacity>

      <TouchableOpacity
        style={styles.qrListBtn}
        onPress={() => setShowQRList(true)}
      >
        <Text style={styles.qrListText}>QR LIST</Text>
      </TouchableOpacity>

      <Modal
        visible={showWalletSelect}
        transparent
        animationType="fade"
        onRequestClose={() => { setShowWalletSelect(false); setScanned(false); }}
      >
        <View style={styles.modalOverlay}>
          <View style={styles.walletModalContent}>
            <Text style={styles.modalTitle}>
              Choose your app:
            </Text>
            <Text style={styles.disclaimerNeutral}>
              UR Scanner is not affiliated with any bank or e-wallet.
            </Text>
            <Text style={styles.merchantName}>
              {currentQR ? extractMerchantName(currentQR) : ''}
            </Text>

            <ScrollView style={{ maxHeight: 400 }}>
              {WALLETS_SORTED.map((wallet) => (
                <TouchableOpacity
                  key={wallet.name}
                  style={[styles.walletBtn, { backgroundColor: wallet.color }]}
                  onPress={() => handleWalletSelect(wallet.name)}
                >
                  <View style={styles.walletBtnContent}>
                    <View style={[styles.walletIcon, { backgroundColor: '#fff' }]}>
                      <Text style={[styles.walletIconText, { color: wallet.color }]}>
                        {wallet.name.charAt(0)}
                      </Text>
                    </View>
                    <Text style={styles.walletBtnText}>{wallet.name}</Text>
                  </View>
                </TouchableOpacity>
              ))}

              <TouchableOpacity
                style={[styles.walletBtn, { backgroundColor: '#555' }]}
                onPress={() => handleWalletSelect('QRPH_Default')}
              >
                <View style={styles.walletBtnContent}>
                  <View style={[styles.walletIcon, { backgroundColor: '#fff' }]}>
                    <Text style={[styles.walletIconText, { color: '#555' }]}>QR</Text>
                  </View>
                  <Text style={styles.walletBtnText}>Other QRPH Apps</Text>
                </View>
              </TouchableOpacity>
            </ScrollView>

            <Text style={styles.legalDisclaimer}>
              *Clicks only. Not confirmed payment. You will choose amount inside your app.
            </Text>

            <TouchableOpacity
              style={styles.closeBtn}
              onPress={() => { setShowWalletSelect(false); setScanned(false); }}
            >
              <Text style={styles.closeBtnText}>Cancel</Text>
            </TouchableOpacity>
          </View>
        </View>
      </Modal>

      <Modal
        visible={showSettings}
        transparent
        animationType="fade"
        onRequestClose={() => setShowSettings(false)}
      >
        <View style={styles.modalOverlay}>
          <ImageBackground
            source={require('./assets/ur-scanner-logo-bg.png')}
            style={styles.modalContent}
            imageStyle={styles.modalBgImage}
            resizeMode="cover"
          >
            <View style={styles.modalDarkOverlay} pointerEvents="none" />
            <View style={styles.modalContentInner}>
            {!showLiveStats && !showAbout && !showPrivacy ? (
              <>
                <PulseText text="Settings" size={20} bold style={styles.centerRow} />

                <TouchableOpacity
                  style={styles.modalItem}
                  onPress={() => setShowLiveStats(true)}
                >
                  <PulseText text="📊 Live Stats" size={16} />
                </TouchableOpacity>

                <TouchableOpacity
                  style={styles.modalItem}
                  onPress={() => setShowPrivacy(true)}
                >
                  <PulseText text="🔒 Privacy Policy" size={16} />
                </TouchableOpacity>

                <TouchableOpacity
                  style={styles.modalItem}
                  onPress={() => setShowAbout(true)}
                >
                  <PulseText text="ℹ️ About UR Scanner" size={16} />
                </TouchableOpacity>

                <Text style={styles.versionText}>Version 1.0.0</Text>

                <TouchableOpacity
                  style={styles.closeBtn}
                  onPress={() => setShowSettings(false)}
                >
                  <PulseText text="Close" size={14} bold style={styles.centerRow} />
                </TouchableOpacity>
              </>
            ) : showLiveStats ? (
              <>
                <PulseText text="Live Stats" size={20} bold style={styles.centerRow} />
                <Text style={styles.statsHeader}>*Clicks only. Not confirmed payment.*</Text>
                <ScrollView style={{ maxHeight: 400, marginVertical: 10 }}>
                  {wavyStats.map((stat, idx) => (
                    <WavyStatRow
                      key={idx}
                      label={stat.label}
                      value={stat.value}
                      color={stat.color}
                    />
                  ))}
                </ScrollView>
                <TouchableOpacity
                  style={styles.closeBtn}
                  onPress={() => setShowLiveStats(false)}
                >
                  <PulseText text="Back" size={14} bold style={styles.centerRow} />
                </TouchableOpacity>
              </>
            ) : showPrivacy ? (
              <>
                <PulseText text="Privacy Policy" size={20} bold style={styles.centerRow} />
                <ScrollView style={{ maxHeight: 400, marginVertical: 10 }}>
                  <Text style={styles.privacyHeader}>UR Scanner Privacy Policy</Text>
                  <Text style={styles.privacyDeveloper}>Developer: EDISON SUCLATAN DAYAGUIT</Text>
                  <Text style={styles.privacyDate}>Last Updated: 2026-01-24</Text>

                  <Text style={styles.privacySubHeader}>1. Information We Collect</Text>
                  <Text style={styles.aboutText}>
                    - QR Code Data: Temporarily processed to extract merchant name only. Full QR data is not stored.{'\n'}
                    - Click Analytics: Which bank or e-wallet you select. No payment amounts recorded.{'\n'}
                    - Device Data: Anonymous crash logs via Firebase Crashlytics.
                  </Text>

                  <Text style={styles.privacySubHeader}>2. How We Use Data</Text>
                  <Text style={styles.aboutText}>
                    - To redirect you to your chosen bank or e-wallet.{'\n'}
                    - To show live statistics.{'\n'}
                    - To fix crashes and improve performance.
                  </Text>

                  <Text style={styles.privacySubHeader}>3. Data We DON'T Collect</Text>
                  <Text style={styles.aboutText}>
                    - Bank account numbers{'\n'}
                    - PINs or passwords{'\n'}
                    - Payment amounts{'\n'}
                    - Personal identity{'\n'}
                    - Location data{'\n'}
                    - Contacts or photos
                  </Text>

                  <Text style={styles.privacySubHeader}>4. Third Parties</Text>
                  <Text style={styles.aboutText}>
                    - Firebase Analytics: Anonymous usage stats.{'\n'}
                    - Firebase Crashlytics: Crash reports only.{'\n'}
                    - No data is sold to advertisers.
                  </Text>

                  <Text style={styles.privacySubHeader}>5. Your Rights</Text>
                  <Text style={styles.aboutText}>
                    - Delete saved QR list anytime in-app.{'\n'}
                    - Uninstall to stop all data collection.{'\n'}
                    - Contact: edisondayags@gmail.com
                  </Text>

                  <Text style={styles.privacySubHeader}>6. Children's Privacy</Text>
                  <Text style={styles.aboutText}>
                    This app is not intended for children under 13.
                  </Text>

                  <Text style={styles.privacySubHeader}>7. Changes</Text>
                  <Text style={styles.aboutText}>
                    We may update this policy. Check this page for the latest version.
                  </Text>

                  <Text style={styles.aboutDivider}>──────────────</Text>
                  <Text style={styles.aboutText}>
                    UR Scanner does not process payments. You are redirected to your chosen bank or e-wallet which has its own privacy policy.
                  </Text>
                  <Text style={styles.aboutText}>
                    {'\n'}Developer: EDISON SUCLATAN DAYAGUIT{'\n'}San Antonio Adtuyon Pangantucan Bukidnon
                  </Text>
                </ScrollView>
                <TouchableOpacity
                  style={styles.closeBtn}
                  onPress={() => setShowPrivacy(false)}
                >
                  <PulseText text="Back" size={14} bold style={styles.centerRow} />
                </TouchableOpacity>
              </>
            ) : (
              <>
                <PulseText text="About UR Scanner" size={20} bold style={styles.centerRow} />
                <ScrollView style={{ maxHeight: 400, marginVertical: 10 }}>
                  <Text style={styles.aboutText}>
                    UR Scanner is a QRPH reader that helps you open QR codes in your preferred bank or e-wallet app.
                  </Text>
                  <Text style={styles.aboutText}>
                    {'\n'}This app does not process payments, store funds, or hold user money. It only redirects to your chosen app.
                  </Text>
                  <Text style={styles.aboutText}>
                    {'\n'}*Clicks only. Not confirmed payment. You will choose amount inside your app.
                  </Text>
                  <Text style={styles.aboutDivider}>──────────────</Text>
                  <Text style={styles.aboutDeveloper}>Developer:</Text>
                  <Text style={styles.aboutName}>EDISON SUCLATAN DAYAGUIT</Text>
                  <Text style={styles.aboutAddress}>San Antonio Adtuyon Pangantucan Bukidnon</Text>
                  <Text style={styles.aboutText}>
                    {'\n'}UR Scanner is not affiliated with any bank or e-wallet.
                  </Text>
                  <Text style={styles.aboutQuote}>
                    {'\n'}Yun lang para maintindihan nyo talaga and mabuhay kayo 3x a day mga ka inspirasyon
                  </Text>
                </ScrollView>
                <TouchableOpacity
                  style={styles.closeBtn}
                  onPress={() => setShowAbout(false)}
                >
                  <PulseText text="Back" size={14} bold style={styles.centerRow} />
                </TouchableOpacity>
              </>
            )}
            </View>
          </ImageBackground>
        </View>
      </Modal>

      <Modal
        visible={showQRList}
        transparent
        animationType="slide"
        onRequestClose={() => setShowQRList(false)}
      >
        <View style={styles.modalOverlay}>
          <View style={styles.listModalContent}>
            <Text style={styles.listTitle}>
              QR LIST - Tap to open
            </Text>

            {savedQRs.length === 0 ? (
              <View style={styles.emptyContainer}>
                <Text style={styles.emptyText}>No saved QR yet</Text>
              </View>
            ) : (
              <FlatList
                data={savedQRs}
                keyExtractor={(_, i) => i.toString()}
                renderItem={({ item, index }) => (
                  <TouchableOpacity
                    style={styles.qrItem}
                    onPress={() => openQR(item.data)}
                  >
                    <View style={styles.qrItemLeft}>
                      <Text style={styles.qrItemName}>{item.name}</Text>
                      <Text style={styles.qrItemDate}>Saved: {item.date.substring(0, 10)}</Text>
                    </View>
                    <TouchableOpacity
                      onPress={() => deleteQR(index)}
                    >
                      <Text style={styles.deleteText}>🗑️</Text>
                    </TouchableOpacity>
                  </TouchableOpacity>
                )}
              />
            )}

            <TouchableOpacity
              style={styles.closeBtn}
              onPress={() => setShowQRList(false)}
            >
              <Text style={styles.closeBtnText}>Close</Text>
            </TouchableOpacity>
          </View>
        </View>
      </Modal>
    </SafeAreaView>
  );
}

function PulseLetter({ char, index, baseSize, bold }) {
  const progress = useSharedValue(0);

  useEffect(() => {
    progress.value = withDelay(
      index * 80,
      withRepeat(
        withSequence(
          withTiming(1, { duration: 900, easing: Easing.inOut(Easing.ease) }),
          withTiming(0, { duration: 900, easing: Easing.inOut(Easing.ease) })
        ),
        -1,
        false
      )
    );
  }, []);

  const animStyle = useAnimatedStyle(() => {
    const color = interpolateColor(
      progress.value,
      [0, 1],
      ['#FF4D8D', '#0A0A0A']
    );
    return { color };
  });

  return (
    <Animated.Text
      style={[
        { fontSize: baseSize || 16, fontWeight: bold ? 'bold' : '600' },
        animStyle,
      ]}
    >
      {char === ' ' ? '\u00A0' : char}
    </Animated.Text>
  );
}

function PulseText({ text, size, bold, style }) {
  return (
    <View
      style={[{ flexDirection: 'row', flexWrap: 'wrap' }, style]}
    >
      {text.split('').map((char, i) => (
        <PulseLetter key={i} char={char} index={i} baseSize={size} bold={bold} />
      ))}
    </View>
  );
}

function WavyStatRow({ label, value, color }) {
  const display = label + ' = ' + value.toLocaleString();
  return (
    <View
      style={styles.wavyRow}
    >
      {display.split('').map((char, i) => {
        const isNum = !isNaN(char) && char !== ' ';
        return (
          <WavyChar key={i} char={char} index={i} textColor={isNum ? color : '#FFFFFF'} />
        );
      })}
    </View>
  );
}

function WavyChar({ char, index, textColor }) {
  const translateY = useSharedValue(0);

  useEffect(() => {
    const animate = () => {
      translateY.value = withSequence(
        withDelay(index * 30, withTiming(-6, { duration: 150 })),
        withTiming(6, { duration: 300 }),
        withTiming(0, { duration: 150 })
      );
    };
    animate();
    const id = setInterval(animate, 5000);
    return () => clearInterval(id);
  }, []);

  const animStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
  }));

  return (
    <Animated.Text
      style={[styles.wavyChar, { color: textColor }, animStyle]}
    >
      {char}
    </Animated.Text>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#000' },
  permText: { color: '#fff', textAlign: 'center', marginTop: 40 },
  settingsBtn: {
    position: 'absolute',
    top: 50,
    left: 20,
    width: 44,
    height: 44,
    borderRadius: 22,
    backgroundColor: 'rgba(0,0,0,0.55)',
    justifyContent: 'center',
    alignItems: 'center',
  },
  iconText: { fontSize: 24 },
  qrListBtn: {
    position: 'absolute',
    top: 50,
    right: 20,
    paddingHorizontal: 14,
    paddingVertical: 9,
    borderRadius: 20,
    backgroundColor: brandPinkDark,
  },
  qrListText: { color: '#fff', fontWeight: 'bold' },
  modalOverlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.72)',
    justifyContent: 'center',
    alignItems: 'center',
  },
  modalContent: {
    width: '85%',
    maxHeight: '80%',
    borderRadius: 12,
    padding: 20,
    overflow: 'hidden',
  },
  modalBgImage: {
    borderRadius: 12,
    opacity: 0.9,
  },
  modalDarkOverlay: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: 'rgba(0,0,0,0.55)',
    borderRadius: 12,
  },
  modalContentInner: {
    position: 'relative',
    zIndex: 1,
  },
  centerRow: {
    justifyContent: 'center',
    marginBottom: 8,
  },
  walletModalContent: {
    width: '85%',
    maxHeight: '80%',
    backgroundColor: '#1a1a1a',
    borderRadius: 12,
    padding: 20,
  },
  modalTitle: {
    color: '#fff',
    fontSize: 20,
    fontWeight: 'bold',
    marginBottom: 8,
    textAlign: 'center',
  },
  merchantName: {
    color: brandPinkLight,
    fontSize: 14,
    textAlign: 'center',
    marginBottom: 8,
  },
  disclaimerNeutral: {
    color: '#AAA',
    fontSize: 11,
    textAlign: 'center',
    marginBottom: 12,
    fontStyle: 'italic',
  },
  legalDisclaimer: {
    color: '#777',
    fontSize: 10,
    textAlign: 'center',
    marginTop: 12,
    fontStyle: 'italic',
  },
  walletBtn: {
    padding: 12,
    borderRadius: 10,
    marginVertical: 5,
    elevation: 2,
  },
  walletBtnContent: { flexDirection: 'row', alignItems: 'center' },
  walletIcon: {
    width: 32,
    height: 32,
    borderRadius: 16,
    justifyContent: 'center',
    alignItems: 'center',
    marginRight: 12,
  },
  walletIconText: { fontWeight: '900', fontSize: 16 },
  walletBtnText: {
    color: '#fff',
    fontWeight: 'bold',
    fontSize: 16,
    textShadowColor: 'rgba(0,0,0,0.3)',
    textShadowOffset: { width: 0, height: 1 },
    textShadowRadius: 2,
  },
  modalItem: { paddingVertical: 12 },
  modalItemText: { color: '#fff', fontSize: 16 },
  versionText: { color: '#999', fontSize: 14, marginTop: 8 },
  closeBtn: {
    marginTop: 16,
    padding: 12,
    backgroundColor: '#333',
    borderRadius: 8,
    alignItems: 'center',
  },
  closeBtnText: { color: '#fff', fontWeight: 'bold' },
  listModalContent: {
    width: '90%',
    height: '70%',
    backgroundColor: '#1a1a1a',
    borderRadius: 12,
    padding: 16,
  },
  listTitle: { color: brandPink, fontSize: 18, fontWeight: 'bold', marginBottom: 16 },
  emptyContainer: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  emptyText: { color: '#999', fontSize: 16 },
  qrItem: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingVertical: 12,
    borderBottomWidth: 1,
    borderBottomColor: '#333',
  },
  qrItemLeft: { flex: 1 },
  qrItemName: { color: brandPinkLight, fontSize: 16, fontWeight: 'bold' },
  qrItemDate: { color: '#999', fontSize: 12, marginTop: 4 },
  deleteText: { fontSize: 24 },
  statsHeader: {
    color: '#FFD600',
    fontSize: 10,
    fontWeight: 'bold',
    marginBottom: 6,
    textAlign: 'center',
  },
  wavyRow: { flexDirection: 'row', marginVertical: 2, flexWrap: 'wrap' },
  wavyChar: { fontSize: 11, fontWeight: '600' },
  aboutText: { color: '#CCC', fontSize: 13, lineHeight: 18, marginBottom: 8 },
  aboutDivider: { color: '#555', fontSize: 12, textAlign: 'center', marginVertical: 12 },
  aboutDeveloper: { color: '#AAA', fontSize: 11, textAlign: 'center', marginBottom: 4 },
  aboutName: {
    color: '#FFF',
    fontSize: 16,
    fontWeight: 'bold',
    textAlign: 'center',
    marginBottom: 4,
  },
  aboutAddress: { color: '#AAA', fontSize: 12, textAlign: 'center', marginBottom: 12 },
  aboutQuote: {
    color: brandPinkLight,
    fontSize: 13,
    fontStyle: 'italic',
    textAlign: 'center',
    marginTop: 12,
    lineHeight: 18,
  },
  privacyHeader: {
    color: '#FFF',
    fontSize: 16,
    fontWeight: 'bold',
    textAlign: 'center',
    marginBottom: 4,
  },
  privacyDeveloper: { color: '#CCC', fontSize: 12, textAlign: 'center', marginBottom: 2 },
  privacyDate: { color: '#AAA', fontSize: 11, textAlign: 'center', marginBottom: 12 },
  privacySubHeader: {
    color: brandPinkLight,
    fontSize: 14,
    fontWeight: 'bold',
    marginTop: 12,
    marginBottom: 6,
  },
});
