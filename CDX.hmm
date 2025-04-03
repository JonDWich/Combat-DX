Code "CDX" by "GordinRamsay"
#region CURRENT BUGS:
/*
So it turns out that a mod like this, which essentially needs to create and run a new script in *every single combat encounter*, is a REALLY bad fit for an HMM code, mainly because I don't have a clean way to do a "clean refresh" of the script after every fight. Or after you die. Or after you reload a save. Or after you go to the title. Or pause behavior when you pause the game. And all of this needs to be able to dynamically swap between 4 completely unique characters, typically *without* resetting everything.
As such, there are some bugs. I've tried to eliminate them when I can, or at least mitigate how often they can occur, but I wasn't able to stomp them all. If you see some hacky solution in this code, it's probably related to my inexperience with C#, plus the aforementioned jank of trying to make this thing work as a code. Anyway, here are some known issues:

- Sometimes, when selecting "return to title", the game crashes. Consider this a sign from the heavens that you've played too much Sonic for the day. (Realistically, there isn't a lot I can do since we currently can't detect pauses via HMM)
- QuestTarget text can get stuck on screen, usually seen as a rank message on top of a Skill Bonus. Pretty sure this is caused by trying to set multiple targets too close to each other, but it's a rather rare bug, and doesn't really affect gameplay. Dying fixes this.
- Sometimes, triggering an Aspect after dying as Super and reattempting the boss climb will crash the game. Dunno what the cause is, have only had it happen once. At a guess, it's trying to trigger an Aspect while you're Super and running into some null value?
- Occasionally you'll get a black screen when starting a Super fight. Opening the tips menu will fix this, but that's not always available. Calling "FadeIn" does sort this, but depending on how you start the fight, Super may not make his Init call (usually if you do Master King's trial then immediately load Supreme).
- Changing difficulty won't actually update difficulty scaling until a full INIT is called. Could fix by running Inits more frequently, but I'm trying to avoid that as it can cause combat to reset if you get an enemy outside of it's territory, even when you're still combo'ing them.
- Cyloop letterboxing occasionally occurs when initiating combat. Need to add additional checks to prevent that, but it's kinda cinematic at the start of a Super Sonic fight, so I'm keeping it for now.
- Parries don't always initiate a skill link, and attacks aren't always added to the Link. Dunno what's causing it; I've tried debugging it, but the game straight up does not detect the animation as having happened in these instances. This miiiight be fixable by detecting States instead of Animations, but there are some advantages to using Anims that I really don't want to give up. Having GuidUI feedback on Skill Links mitigates this anyway.
- Super's natural Phantom Rush regeneration sometimes just doesn't work after dying. Dunno why, and it does seem to naturally fix itself after a bit, but I still have no idea why. 
- The game will crash if you parry Supreme Beast's "The Voices" attack. This doesn't seem to be a crash from C# though, so it may not be related to CDX.
*/
#endregion
#region TO DO:
/*
The mod's pretty much functional, but there's still some stuff I need to add later.
- Skill Links and Aspects for the Amigos. Might have to accompany it with a bigger Amigo rework.
- Touch up the Rifle Beast fight.
- Tweak aspects a bit? Tails feels kind of underwhelming. Maybe make Homing Shot drag in enemies.
- Improve Init checking. Check if playerName ~= previous playerName then force an Init? 
*/
#endregion
//
	#lib "HMM"
	#lib "INI"
	#lib "Player"
	#lib "SonicParameters"
	#lib "AmyParameters"
	#lib "KnucklesParameters"
	#lib "TailsParameters"
	#lib "Lua"
	#lib "Time"
	#include "BlackboardStatus" noemit
	#lib "BlackboardItem"
	#lib "BlackboardBattle"
	#lib "BlackboardStatus"
	#lib "XInput"
	#lib "Converse"
	#include "Reflection" noemit
	#include "ReflectionHelpers" noemit
	#lib "Reflection"
	#lib "GameHitStopParameter"
	
	#import "Plugins"
	#import "States"
	
	#import "Messages"
	
	using System.Collections.Generic;
	using System.Numerics;
	
	public static Dictionary<string,int> MoveIndex = new()
	{
		//This is indexed using the return from GetCurrentAnimationName, which is in SCREAMING_SNAKE_CASE.
		{"PARRY", 0}, //Common & Sonic anims
		{"PARRY_AIR", 0},
		{"PARRY_JUST", 0},
		{"COMBO_SONICBOOM", 1},
		{"COMBO_CROSSSLASH", 2},
		{"COMBO_LOOPKICK_START", 3},
		{"COMBO_HOMINGSHOT", 4},
		{"STOMPING_START", 5},
		{"STOMPING", 5},
		{"COMBO_CHARGE", 6},
		{"COMBO_CHARGE_LOOP", 6},
		{"COMBO_CHARGE_FINISH", 6},
		{"COMBO_CHARGE_END", 6},
		{"COMBO_CRASHER_START", 7},
		{"COMBO_CRASHER_LOOP", 7},
		{"COMBO_SMASH_START", 8},
		{"COMBO_SMASH_LOOP", 8},
		{"COMBO_SMASH", 8},
		{"COMBO_ACCELE_PUNCH01", 9},
		{"COMBO_ACCELE_PUNCH05", 9},
		{"COMBO_ACCELE_KICK01", 9},
		{"COMBO_ACCELE_KICK05", 9},
		{"SLIDING_LOOP", 10},
		{"COMBO_FINISH", 11},
		{"COMBO_FINISH_ACCELE", 11},
		{"COMBO_FINISH_F", 12},
		{"COMBO_FINISH_ACCELE_F", 12},
		{"COMBO_FINISH_B", 13},
		{"COMBO_FINISH_ACCELE_B", 13},
		{"COMBO_FINISH_L", 14},
		{"COMBO_FINISH_ACCELE_L", 14},
		{"COMBO_FINISH_R", 15},
		{"COMBO_FINISH_ACCELE_R", 15},
		{"COMBO_PURSUIT", 16}, 
		//17 is used for Recovery Smash
		{"Cyloop", 18},//This one isn't called by an animation name, I'm passing it manually.
		{"ATK_TAROT", 1}, //Amy anims
		{"ATK_TAROT_AIR", 1},
		{"ATK_TAROT02", 2},
		{"ATK_TAROT02_AIR", 2},
		{"ATK_TAROT_ROLL_START", 3},
		{"RING_MAX_START", 3},
		{"THROW_KISS", 4},
		//Stomping takes 5
		{"CYHAMMER", 6},
		{"CYHAMMER_AIR_START", 6},
		{"COMBO_PUNCH01", 1}, //Knuckles anims
		{"COMBO_PUNCH01_AIR", 1},
		{"COMBO_PUNCH02", 2},
		{"COMBO_PUNCH02_AIR", 2},
		{"COMBO_UPPERCUT", 3},
		{"HEAT_KNUCKLE_START", 4},
		//Stomping takes 5
		{"CYKNUCKLE", 6},
		{"CYKNUCKLE_AIR_LOOP", 6},
		{"ATK_SPANNER_START", 1}, //Tails anims
		{"ATK_SPANNER_ARCH", 1},
		{"ATK_SPANNER_FLY_START", 2}, //Honestly this isn't really different, but I need some padding.
		{"ATK_SPANNER_FLY_ARCH", 2},
		{"ATK_SPANNER_FLOAT", 3},
		{"CYBLASTER_SHOT", 4}
		//Stomping takes 5
	}
	public static Dictionary<string,string> ReactionIndex = new()
	{
		{"BATTLE_DAMAGE", "Damage"}, //Damage
		{"DAMAGE_END", "Damage"}, //Used for Super
		{"DAMAGE02", "Damage"}, //Used for Super
	    {"BATTLE_DAMAGE_BLOW_FRONT", "HeavyKB"}, //HeavyKB
	    {"AVOID_FRONT_AIR", "AirDodge"}, //AirDodge
	    {"AVOID_BACK_AIR", "AirDodge"},
	    {"AVOID_LEFT_AIR", "AirDodge"},
	    {"AVOID_RIGHT_AIR", "AirDodge"},
	    {"PARRY_MISS", "MissedParry"}, //MissedParry
	    {"PARRY_MISS_AIR", "MissedParry"},
		//{"STAND", "COMBAT_EXIT"}, //Player is 100% not in combat. Disabled Stand because it can proc in the air.
		{"RUNNING", "COMBAT_EXIT"},
		{"DEAD", "COMBAT_EXIT"},
		{"DEAD_LOOP", "COMBAT_EXIT"}
	}
	public class MoveBase
	{
	  public int Counter;
	  public int[] PointBonus = new int [5];
	  public MoveBase(int v1, int v2, int v3, int v4, int v5)
	  {
		PointBonus = new int[] {v1, v2, v3, v4, v5};
		Counter = 0;
	  }
	}
	public class SkillBonus
	{
		public string Effect;
		public string SubEffect;
		public float Potency;
		public float Length;
		
		public SkillBonus(string in_effect, string in_subEffect, float in_potency, float in_length)
		{
			Effect = in_effect;
			SubEffect = in_subEffect;
			Potency = in_potency;
			Length = in_length;
		}
	}
	public static class Utilities
	{
		public static void UIToggle(string uiType, string uiElement, string uiMessage = "", string uiHeader = "")
		{
			switch (uiType)
			{
				case "LetterBox":
					if (uiElement != "reset" && ReactionIndex.ContainsKey(currentAnim))
						uiElement = ReactionIndex[currentAnim];
					switch (uiElement)
					{
						case "MissedParry": case "Damage": case "HeavyKB": case "reset":
							Lua.Call("EndLetterBoxUI");
							letterBoxTimer = 0.0f;
							return;
						default:
							letterBoxTimer = 3.0f;
							Lua.Call("StartLetterBoxUI");
							return;
					}
					return;
				case "GuidUI":
					Lua.Call("ClearUserGuidUI");
					//Start UI timer
					switch (uiElement)
					{
						case "reset":
							isGuidUI = false;
							//End timer
							return;
						case "enable":
							//Enable timer?
							isGuidUI = true;
							Converse.Redirect("tu2000_110", uiMessage);
							Lua.Call("ShowUserGuidUI", "Cyloop");
							return;
					}
					return;
				case "QuestTarget":
					switch (uiElement)
					{
						case "reset":
							isRankUI = false;
							Lua.Call("ClearQuestTarget");
							return;
						case "enable":
							isRankUI = true;
							Converse.Redirect("tu2000_030", uiMessage);
							Lua.Call("ShowQuestTarget", "tu2000_030");
							return;
					}
					return;
				case "HeaderWindow":
					Converse.Redirect("tu1000_010", uiMessage);
					Converse.Redirect("tu1000_015", uiHeader);
					Lua.Call("ShowHeaderWindowUI", "tu1000_015", "tu1000_010");
					return;
			}
		}
		public static float GetBattleInformation(string combatStat)
		{
			var pBlackboardBattle = BlackboardBattle.Get();
			if (pBlackboardBattle == null)
				return 0.0f;
			switch (combatStat)
			{
				case "Cyloop":
					return pBlackboardBattle->QuickCyloopAmount;
				case "PhantomRush":
					return pBlackboardBattle->PhantomRushAmount;
				case "Combo":
					return pBlackboardBattle->ComboCount;
			}
			return 0.0f;
		}
		public static void IsRushActive()
		{
			var pStatePluginBattle = Player.State.GetStatePlugin<StatePluginBattle>();
			if (pStatePluginBattle == null)
				return;
			isRush = pStatePluginBattle->Flags.HasFlag(StatePluginBattle.BattleFlags.IsPhantomRush);
		}
		public static void SetBattleInformation(string combatStat, float val)
		{
			var pBlackboardBattle = BlackboardBattle.Get();
			switch (combatStat)
			{
				case "Cyloop":
					pBlackboardBattle->QuickCyloopAmount += val;
					return;
				case "PhantomRush":
					pBlackboardBattle->PhantomRushAmount += val;
					return;
			}
		}
		public static void HoldOnOff(bool isHold, float holdTime = 0.0f)
		{
			var pPlayer = Player.GetPlayerData();
			if (pPlayer == null || Player.GetPlayerType() == null)
				return;
			if (isHold)
			{
				holdTimer = holdTime;
				Messages.SendMessageToMessenger<MsgHoldOn>(&pPlayer->GameObject, new MsgHoldOn());
			}
			else
			{
				Messages.SendMessageToMessenger<MsgHoldRelease>(&pPlayer->GameObject, new MsgHoldRelease());
			}			
		}
		public static void DiscardStateByTime(dynamic stateName, float discardTime)
		{
			if (isStateDiscarded)
			{
				Console.WriteLine("RESTORED STATE: " + discardedState);
				Player.State.Restore(discardedState);
				isStateDiscarded = false;
			}
			stateTimer = discardTime;
			discardedState = stateName;
			isStateDiscarded = true;
			Player.State.Discard(stateName);
		}
		public static float ToDecimal(float simplify, int precision)
		{
			float result = (float)Math.Floor(simplify * (float)Math.Pow(10.0f, precision))/(float)Math.Pow(10.0f, precision);
			return result;
		}
	}
	public static class Bonuses
	{
		public static bool bonusOn = false;
		public static bool isPause = false;
		//public static string bonusType; //Used for duration type bonuses.
		//public static float bonusTimer; //^, compares against the bonusDuration
		public static float bonusDuration; //Length of a time based bonus
		public static float pauseVal; //During Meter Lock, hold Phantom Rush at this value
		public static float bonusCritRate;
		public static float bonusCritDamage;
		public static void SetupBonus(string bName, string bMulti = "", float bStr = 0.0f, float bDur = 0.0f)
		{
			if (bName == "Reset") //This needs to execute before anything else 
			{
				Console.WriteLine("Resetting combo....");
				bonusOn = false;
				if (playerName != "Super")
				{
					isPause = false;
					pauseVal = 0.0f;
				}
				Utilities.UIToggle("QuestTarget", "reset");
				Scoring.StyleRankCall(CharacterList[playerName].myRanks);
				//bonusType = "";
				bonusDuration = 0.0f;
				Scoring.rushMultiplier = Scoring.base_RushMultiplier
				Scoring.scoreMultiplier = Scoring.base_ScoreMultiplier;
				bonusCritRate = 0f;
				bonusCritDamage = 0f;
				Aspects.SetCritByBonus();
				return;
			} else if (bonusOn && bName == "Duration") //Only need to return when stacking multipliers. Single/Items are fine to go through.
			{
				uiTimer += 3f;
				Console.WriteLine("____BONUS STILL ACTIVE");
				Utilities.UIToggle("GuidUI", "reset");
				Utilities.UIToggle("GuidUI", "enable", "Bonus already active!!!");
				return;
			}
			float bAdditional = (bStr * bDur) * 10.0f; //Strength for Item/Single bonuses
			switch (bName)
			{
				case "Duration":
					float displayStr;
					Utilities.UIToggle("QuestTarget", "reset");
					Utilities.UIToggle("GuidUI", "reset");
					//bonusType = bName;
					bonusOn = true;
					if (bDur < 7f) //Bonuses don't feel like they last long enough, hopefully this helps.
						bDur = 7f;
					//uiTimer += bDur; Used for GuidUI, probably an old implementation
					switch (bMulti)
					{
						case "Style":
							Console.WriteLine("___ADDED COMBO!!");
							Utilities.UIToggle("QuestTarget", "enable", $"Style Multiplier! +{Math.Floor((bStr * 100) - 100)}%");
							Console.WriteLine(Scoring.scoreMultiplier);
							Scoring.scoreMultiplier = bStr;
							Console.WriteLine("New multi: " + Scoring.scoreMultiplier);
							break;
						case "Rush":
							Utilities.UIToggle("QuestTarget", "enable", $"Rush Multiplier! +{Math.Floor((bStr * 100) - 100)}%");
							Scoring.rushMultiplier = bStr;
							break;
						case "StyleRush":
							float strength = Math.Max(1.2f, (bStr * 0.75f));
							Utilities.UIToggle("QuestTarget", "enable", $"Style &amp; Rush Multiplier! +{Math.Floor((strength * 100) - 100)}%");
							Scoring.rushMultiplier = strength;
							Scoring.scoreMultiplier = strength;
							break;
						case "Cyloop":
							Utilities.UIToggle("QuestTarget", "enable", $"Cyloop Meter Multiplier! +{Math.Floor((bStr * 100) - 100)}%");
							Scoring.CyloopMultiplier = bStr;
							break;
						case "CriticalDamage":
							Console.WriteLine("bStr: " + bStr);
							bStr /= 2f;
							Console.WriteLine("bStr Now: " + bStr);
							//NOTE: Representing this is a bit weird due to how crit damage and rate is handled normally.
							displayStr = (float)Math.Floor(bStr * 100); //The regular formlua doesn't work well since bStr will almost always be less than 1, so I'm trying this.
							Utilities.UIToggle("QuestTarget", "enable", $"Critical Damage Multiplier! +{displayStr}%");
							bonusCritDamage = bStr;
							Aspects.SetCritByBonus();
							break;
						case "CriticalRate":
							Console.WriteLine("bStr: " + bStr);
							bStr /= 2f;
							Console.WriteLine("bStr Now: " + bStr);
							displayStr = (float)Math.Floor(bStr * 100);
							Utilities.UIToggle("QuestTarget", "enable", $"Critical Strike Chance Up! +{displayStr}%");
							bonusCritRate = bStr;
							break;
						case "CriticalAll":
							bStr /= 2f;
							displayStr = (float)Math.Floor(bStr * 100);
							Utilities.UIToggle("QuestTarget", "enable", $"Crit Damage &amp; Rate Up! +{displayStr}%");
							bonusCritDamage = bStr;
							bonusCritRate = bStr;
							Aspects.SetCritByBonus();
							break;
					}
					bonusDuration = bDur;
					return;
				case "Single":
					uiTimer += 3.0f;
					Console.WriteLine($"UI TIMER: {uiTimer}");
					float currentGauge = Utilities.GetBattleInformation("PhantomRush");
					float currentLoop = Utilities.GetBattleInformation("Cyloop"); 
					switch (bMulti)
					{
						case "Rush":
							Utilities.UIToggle("GuidUI", "enable", $"Rush Bonus! +{bAdditional}");
							Lua.Call("SetPhantomRushGauge", currentGauge + bAdditional);
							break;
						case "Style":
							Utilities.UIToggle("GuidUI", "enable", $"Style Bonus! +{bAdditional}");
							Scoring.styleScore += (int)bAdditional;
							break;
						case "StyleRush":
							float strength = bAdditional * 0.75f;
							Utilities.UIToggle("GuidUI", "enable", $"Style &amp; Rush Bonus! +{strength}");
							Lua.Call("SetPhantomRushGauge", currentGauge + strength);
							Scoring.styleScore += (int)strength;
							break;
						case "Cyloop":
							Utilities.UIToggle("GuidUI", "enable", $"Cyloop Bonus! +{bAdditional}");
							Lua.Call("SetQuickCyloopGauge", (float)Math.Min(120.0f, currentLoop + bAdditional));
							break;
					}
					return;
				case "Item":
					uiTimer += 3.0f;
					//Normally I'd use a Switch, but Ring is all I'm adding atm
					bAdditional = (float)Math.Ceiling(bAdditional);
					Utilities.UIToggle("GuidUI", "enable", $"Ring bonus! +{bAdditional} rings!");
					Lua.Call("PlayerGetItem", "Ring", bAdditional);
					return;
				case "Unlimited":
					//bonusType = bName;
					Utilities.UIToggle("GuidUI", "reset");
					bonusOn = true;
					bonusDuration = bDur;
					isPause = true;
					pauseVal = 120f;
					Utilities.UIToggle("QuestTarget", "enable", "Unlimited Phantom Rush!!!");
					return;
				case "Pause":
					//bonusType = "Duration";
					Utilities.UIToggle("GuidUI", "reset");
					bonusDuration = bDur;
					bonusOn = true;
					isPause = true;
					pauseVal = Utilities.GetBattleInformation("PhantomRush"); 
					Utilities.UIToggle("QuestTarget", "enable", $"Meter Locked at {Math.Floor(pauseVal)}%!");
					return;
			}
		}
	}
	public static class SkillLink
	{
		public static int ChainLevel = 1;
		public static bool ChainActive = false;
		public static List<string> ComboBonus = new();
		public static Dictionary<string,string> ComboStringCreate = new()
		{
			{"PARRY", "Parry"},
			{"PARRY_AIR", "Parry"},
			{"COMBO_SONICBOOM", "SonicBoom"},
			{"COMBO_CROSSSLASH", "CrossSlash"},
			{"COMBO_LOOPKICK_START", "LoopKick"},
			{"COMBO_HOMINGSHOT", "HomingShot"},
			{"STOMPING", "Stomp"},
			{"COMBO_CHARGE_LOOP", "CycloneKick"},
			{"COMBO_CRASHER_START", "Crasher"},
			{"COMBO_SMASH_START", "Smash"},
			{"COMBO_SMASH_LOOP", "Smash"},
			{"COMBO_SMASH", "Smash"},
			{"COMBO_ACCELE_KICK01", "Standard"},
			{"COMBO_ACCELE_KICK05", "Standard"},
			{"COMBO_ACCELE_PUNCH01", "Standard"},
			{"COMBO_ACCELE_PUNCH05", "Standard"},
			{"COMBO_PURSUIT", "Standard"},
			{"SLIDING_LOOP", "Standard"},
			{"Cyloop", "Cyloop"}
		}
		public static bool SpecialCombo(string comboString)
		{
			switch(comboString)
			{
				case "Parry Parry Parry":
					//Set bonus for Unlimited Phantom Rush
					Utilities.UIToggle("QuestTarget", "reset");
					Bonuses.SetupBonus("Unlimited", "_", 0, 15);
					return true;
				case "Smash Smash Smash":
					//Set bonus for Meter Lock
					Utilities.UIToggle("QuestTarget", "reset");
					Bonuses.SetupBonus("Pause", "_", 0, 10);
					return true;
				case "CycloneKick Standard SonicBoom":
					//Max the Cyloop gauge
					Lua.Call("SetQuickCyloopGauge", 120f);
					//Might need to reset GuidUI here? Works for now, just keep in mind if things break later.
					Utilities.UIToggle("GuidUI", "enable", "Cyloop Gauge Maxed!!");
					uiTimer += 3f;
					return true;
				/*case "Cyloop Crasher Stomp": I just didn't want to implement these.
					//OneTime Rush bonus?
					return true;
				case "Cyloop CrossSlash LoopKick":
					//OneTime Style bonus?
					return true;
				case "Cyloop SonicBoom Stomp":
					//Max the Cyloop gauge
					return true;*/
			}
			return false;
		}
		public static void ComboCall(string move)
		{
			if (ComboStringCreate.ContainsKey(move))
			{
				move = ComboStringCreate[move];
				if (ComboBonus.Count < 3)
				{
					ComboBonus.Add(move);
					//Player.Sound.PlaySound("_sn_airtrick");
					Console.WriteLine("Added ability: " + move);
					int cCounter = ComboBonus.Count;
					switch (cCounter)
					{
						case 1:
							Utilities.UIToggle("GuidUI", "enable", $"Skill Link: {ComboBonus[0]} | ??? | ???");
							break;
						case 2:
							Utilities.UIToggle("GuidUI", "enable", $"Skill Link: {ComboBonus[0]} | {ComboBonus[1]} | ???");
							break;
						case 3:
							//Utilities.UIToggle("GuidUI", "enable", $"Skill Link: {ComboBonus[0]} | {ComboBonus[1]} | {ComboBonus[2]}");
							break;
					}
				}
			}
			if (ComboBonus.Count >= 3)
			{
				if (ComboBonus[2] == "Cyloop" && ChainLevel < 3)
				{
					Console.WriteLine("Chain level up!!!!!!");
					ChainLevel += 1;
					Utilities.UIToggle("GuidUI", "enable", $"Skill Link level up! Level {ChainLevel}/3");
					if (Scoring.CyloopCounter < ChainLevel)
					{
						Console.WriteLine("Cyloop level up!!!");
						Scoring.CyloopCounter += 1;
					}
				}
				else
				{
					ChainActive = false;
					string comboString = String.Join(" ", ComboBonus);
					if (!SpecialCombo(comboString))
					{
						var myChar = CharacterList[playerName];
						string eff = myChar.MoveBonuses[ComboBonus[0]][ChainLevel - 1].Effect;
						string subEff = myChar.MoveBonuses[ComboBonus[0]][ChainLevel - 1].SubEffect;
						float potency = myChar.MoveBonuses[ComboBonus[1]][ChainLevel - 1].Potency;
						float duration = myChar.MoveBonuses[ComboBonus[2]][ChainLevel - 1].Length;
						if (eff == "Duration" || eff == "Unlimited" || eff == "Pause")
						{
							switch (ChainLevel)
							{
								case 1:
									duration = Math.Min(7.0f, duration);
									break;
								case 2:
									duration = Math.Min(12.0f, duration);
									break;
								case 3:
									duration = Math.Min(18.0f, duration);
									break;
							}
						} 
						Console.WriteLine("Making bonus with: " + ComboBonus[0] + ComboBonus[1] + ComboBonus[2]);
						Console.WriteLine($"EFF: {eff}, SUB: {subEff}, POT: {potency}, DUR: {duration}");
						Bonuses.SetupBonus(eff, subEff, potency, duration);
					}
					ChainLevel = 1;
				}
				//uiTimer += 1.5f; //Countdown for disabling the GuidUI
				ComboBonus.Clear();
			}
		}
	};
	public class RankLevel
	{
		public string rankID;
		public int scoreReq;
		public int rankLevel;
		public string voiceLine;
		public int rankLockout;
		public bool hasPlayed;
		public string questTarget = "Combo Rank: ";
		public RankLevel(string rankName, int scoreCost, int rankLvl, string announcerLine, int lockoutTime = 4)
		{
			rankID = rankName;
			scoreReq = scoreCost;
			rankLevel = rankLvl;
			voiceLine = announcerLine;
			rankLockout = lockoutTime;
			questTarget += rankName;
			hasPlayed = false;
		}
	};
	public class TauntBase
	{
		public string tauntAnim;
		public string tauntVoice;
		public float tauntDuration;
		public bool useLuaVoice; //PlayVoice only works for certain sound banks, so HMM has to be used for the others.
		public TauntBase(string tName, string tVoice, float tDur, bool isLua = true)
		{
			tauntAnim = tName;
			tauntVoice = tVoice;
			tauntDuration = tDur;
			useLuaVoice = isLua;
		}	
	};
	public struct Aspects
	{
		public static float aspectBonusCrit = 0f;
		public static float aspectBonusCritDamage = 0f;
		public static float baseCritDamage = 3f;
		public static void OnParryAspect(string aspect)
		{
			var SonicParams = Reflection.GetDataInfo<SonicParameters.Root>("player_common");
			var minDamage = RFL_GET_PARAM(SonicParams, common.commonPackage.attack.common.offensive.pointMin);
			var maxDamage = RFL_GET_PARAM(SonicParams, common.commonPackage.attack.common.offensive.pointMax);
			var critRate = RFL_GET_PARAM(SonicParams, common.commonPackage.attack.common.criticalRate);	
			var critDamageRate = RFL_GET_PARAM(SonicParams, common.commonPackage.attack.common.criticalRate);			
			Console.WriteLine("MIN: " + minDamage + " MAX: " + maxDamage + " CRIT: " + critRate + " | " + critDamageRate);
			switch (aspect)
			{
				case "Amy":
					if (minDamage < maxDamage)
					{
						RFL_SET_PARAM(SonicParams, common.commonPackage.attack.common.offensive.pointMin, (ushort)(minDamage + 1));
					}
					break;
				case "Knuckles":
					if (maxDamage < 10)
					{
						//RFL_SET_PARAM(SonicParams, common.commonPackage.attack.common.offensive.pointMin, (ushort)(minDamage + 1));
						RFL_SET_PARAM(SonicParams, common.commonPackage.attack.common.offensive.pointMax, (ushort)(maxDamage + 1));
					}
					break;
				case "Sonic":
					aspectBonusCrit += 0.1f;
					SetCritByTension();
					//RFL_SET_PARAM(SonicParams, common.commonPackage.attack.common.criticalRate, (float)(critRate + 0.1f));
					break;
				case "Tails":
					Lua.Call("PlayerGetItem", "Ring", 25);
					aspectBonusCritDamage += 0.15f;
					SetCritByBonus();
					break;
			}
		}
		public static void ReduceDamageMultiplier(float damageReduction = 1f)
		{
			var SonicParams = Reflection.GetDataInfo<SonicParameters.Root>("player_common");
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.homingAttack.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.homingAttackAir.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.pursuitKick.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.stomping.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.boundStompingLast.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.sliding.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.loopKick.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crasher.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.spinSlashLast.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.sonicBoom.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crossSlash.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.homingShot.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.chargeAttack.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.chargeAttackLast.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.cyloop.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.cyloopQuick.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.cyloopAerial.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.accele1.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.accele2.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.aerialAccele1.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.aerialAccele2.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.comboFinish.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.comboFinishF.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.comboFinishB.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.comboFinishL.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.comboFinishR.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.acceleComboFinish.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.acceleComboFinishF.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.acceleComboFinishB.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.acceleComboFinishL.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.acceleComboFinishR.damageRateManual, damageReduction);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.smashLast.damageRateManual, damageReduction);
		}
		public static void SetRushDrain(float rate, float rateAccele)
		{
			hasChangedRush = true;
			var SonicParams = Reflection.GetDataInfo<SonicParameters.Root>("player_common");
			if (SonicParams == null)
				return;
			RFL_SET_PARAM(SonicParams, common.commonPackage.acceleMode.declineSpeed, rate);
			RFL_SET_PARAM(SonicParams, common.commonPackage.acceleMode.declineSpeedAccele, rateAccele);
		}
		public static void SetCritByTension()
		{
			var SonicParams = Reflection.GetDataInfo<SonicParameters.Root>("player_common");
			int comboCount = (int)Utilities.GetBattleInformation("Combo");
			float tensionCrit = (comboCount/7.5f)/100f;
			float tensionCritDamage = comboCount/100f;
			if (playerName == "Super")
				RFL_SET_PARAM(SonicParams, common.commonPackage.attack.common.criticalDamageRate, Math.Min(12f, baseCritDamage + tensionCritDamage + aspectBonusCritDamage + Bonuses.bonusCritDamage));
			else
				RFL_SET_PARAM(SonicParams, common.commonPackage.attack.common.criticalRate, Math.Min(0.99f, tensionCrit + aspectBonusCrit + Bonuses.bonusCritRate));
		}
		public static void SetCritByBonus()
		{
			var SonicParams = Reflection.GetDataInfo<SonicParameters.Root>("player_common");
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.common.criticalDamageRate, Math.Min(4f, baseCritDamage + aspectBonusCritDamage + Bonuses.bonusCritDamage));
		}
	}
	public static class Scoring
	{
		//Config settings//
		public static bool usePostBanter; //Unused for now, will add this later when I can find post-battle lines for the Amigos.

		public static string currentDifficulty; //Set in InitVars()
		
		public static int styleScore;
		public static int maxStyleScore;
		public static int comboRank;
		public static int comboLockout;
		public static float difficultyMultiplier;
		
		public static float rushMultiplier = 1f;
		public static float single_RushMultiplier = 1f;
		public static float base_RushMultiplier = 1f;
		
		public static float scoreMultiplier = 1f;
		public static float single_ScoreMultiplier = 1f;
		public static float base_ScoreMultiplier = 1f;
		public static float aspectScoreMultiplier = 1f;
		
		public static float CyloopMultiplier = 1f;
		public static float CyloopGauge = Utilities.GetBattleInformation("Cyloop");
		public static int CyloopCounter = 1; //Number of times you can use QCL
		public static int CyloopUses = 0; //Number of times you have used QCL, relative to Uses
		
		private static float d_ScalingEasy = 1.2f;
		private static float d_ScalingNormal = 1.0f;
		private static float d_ScalingHard = 0.8f;
		private static float r_ScalingEasy = 200f;
		private static float r_ScalingNormal = 300f;
		private static float r_ScalingHard = 400f;
		
		public static void InitVars()
		{
			styleScore = 0;
			maxStyleScore = 0;
			comboRank = 0;
			comboLockout = 0;
			
			//rushMultiplier = 1f;
			single_RushMultiplier = 1f;
			//base_RushMultiplier = 1f;
			
			scoreMultiplier = 1f;
			single_ScoreMultiplier = 1f;
			base_ScoreMultiplier = 1f;
			aspectScoreMultiplier = 1f;
			
			CyloopMultiplier = 1f;
			CyloopGauge = Utilities.GetBattleInformation("Cyloop");
			CyloopCounter = 1;
			CyloopUses = 0;
			var pBlackboardStatus = BlackboardStatus.Get();
			if (pBlackboardStatus == null)
			{
				//If this check returns null for whatever reason, I still need to init. So we add a quick timer and 
				//just run it again when it's definitely going to work.
				failSafeInitTimer = 1.5f;
				return;
			}
			currentDifficulty = pBlackboardStatus->Difficulty.ToString();
			Console.WriteLine("DIFFICULTY: " + currentDifficulty);
			int atkLevel = Lua.Call<int>("GetPowerLevel");			
			switch (currentDifficulty)
			{
				case "Easy":
					superRushScaling = 15f;
					difficultyMultiplier = d_ScalingEasy;
					base_RushMultiplier = Utilities.ToDecimal(1 + (atkLevel/r_ScalingEasy), 2); //1.49x
					break;
				case "Normal":
					superRushScaling = 25f;
					difficultyMultiplier = d_ScalingNormal;
					base_RushMultiplier = Utilities.ToDecimal(1 + (atkLevel/r_ScalingNormal), 2); //1.32x
					break;
				case "Hard": case "Extreme":
					superRushScaling = 35f;
					difficultyMultiplier = d_ScalingHard;
					base_RushMultiplier = Utilities.ToDecimal(1 + (atkLevel/r_ScalingHard), 2); //1.24x
					break;
			}
			rushMultiplier = base_RushMultiplier;
			Console.WriteLine("RUSH MULTIPLIER: " + rushMultiplier);
			
			/* CONFIG INIT */
			
			//So there's this hilarious thing where nothing actually inits. For some reason. Isn't that quirky
			/*var mod = HMM.GetModByID("E81E407A");
			if (mod == null)
			{
				//Console.WriteLine("MOD IS NULL");
				//return;
			}
			var modConfigIniPath = System.IO.Path.Combine(mod.Path, "config.ini");
			if (!System.IO.File.Exists(modConfigIniPath))
			{
				//Console.WriteLine("FILE DOES NOT EXIST");
				//return;
			}
			var ini = INI.Read(modConfigIniPath);
			if (ini == null)
			{
				//Console.WriteLine("INI DOES NOT EXIST");
				//return;
			}
			useRankUI = INI.Parse<bool>(ini["Presentation"]["RankUI"], useRankUI);
			useRankUI_Super = INI.Parse<bool>(ini["Presentation_Super"]["SuperRankUI"], useRankUI_Super);
			useLetterbox = INI.Parse<bool>(ini["Presentation"]["Letterboxing"], useLetterbox);
			useLetterbox_Super = INI.Parse<bool>(ini["Presentation_Super"]["SuperLetterboxing"], useLetterbox_Super);
			useAnnouncer = INI.Parse<bool>(ini["Presentation"]["Announcer"], useAnnouncer);
			useAnnouncer_Super = INI.Parse<bool>(ini["Presentation_Super"]["SuperAnnouncer"], useAnnouncer_Super);
			Console.WriteLine("RUI: " + useRankUI);
			Console.WriteLine("RUI_S: " + useRankUI_Super);
			Console.WriteLine("LB: " + useLetterbox);
			Console.WriteLine("LB_S: " + useLetterbox_Super);
			Console.WriteLine("ANN: " + useAnnouncer);
			Console.WriteLine("ANN_S: " + useAnnouncer_Super);*/
		}
		public static int UpdateStale(string moveName, ref List<MoveBase> moveRef)
		{
		  int moveNum = MoveIndex[moveName];
		  int styleBonus = moveRef[moveNum].PointBonus[moveRef[moveNum].Counter];
		  Console.WriteLine("------------------");
		  Console.WriteLine("MOVE: " + moveName);
		  Console.WriteLine("SCORE: " + styleBonus);
		  for (int i = 0; i < moveRef.Count - 1; i++)
		  {
			if (i != moveNum)
			{
			  if (moveRef[i].Counter > 0)
			  {
				moveRef[i].Counter -= 1;
			  }
			}
			else
			{
			  if (moveRef[i].Counter < 4)
			  {
				moveRef[i].Counter += 1;
			  }
			}
		  };
		  if (SkillLink.ChainActive)
		  {
			  SkillLink.ComboCall(moveName);
		  } 
		  else
		  {
			  if (moveNum == 0 && CharacterList[playerName].hasSkillLink)
			  {
				  SkillLink.ChainActive = true;
				  Utilities.UIToggle("GuidUI", "enable", "Skill Link: ??? | ??? | ???");
			  }
		  }
		  if (moveNum == 0)
		  {
			  Aspects.OnParryAspect(CharacterList[playerName].currentAspect);
		  }
		  return styleBonus;
		}
		public static void StyleRankCall(List<RankLevel> Ranks)
		{
			if (Bonuses.bonusOn)
				return;
			//Console.WriteLine("STYLE IS CURRENTLY: " + styleScore);
			foreach (var rank in Ranks)
			{
				if (styleScore >= rank.scoreReq)
				{
					if (!rank.hasPlayed)
					{
						rank.hasPlayed = true;
						Lua.Call("PlayVoice", rank.voiceLine);
					}
					if (comboRank != rank.rankLevel)
					{
						Lua.Call("ClearQuestTarget");
						Utilities.UIToggle("QuestTarget", "enable", rank.questTarget);
						comboRank = rank.rankLevel;
						comboLockout = rank.rankLockout;
					}
					break;
				}
			}
		}
	}
	public abstract class CharacterBase
	{
		public Random rnd = new Random();
		public int tauntIndex = 0;
		public string currentAspect = "";
		public bool hasTaunt = false; //Enables Taunts
		public bool hasAspect = false; //Enables toggling Aspects
		public bool hasSkillLink = false; //Enables Skill Links
		public bool hasSetRush = false; //Used for Super Sonic
		public bool hasInitChar = false; //Currently only used for Super, might be useful later.
		public List<TauntBase> Taunts = new();
		public List<RankLevel> myRanks = new();
		public List<MoveBase> Moves = new();
		public Dictionary<string, List<SkillBonus>> MoveBonuses = new();
		public Dictionary<string, List<string>> soundList = new();
		public Dictionary<string, bool> warningList = new(); //Currently only for Super, used to display additional popups.
		public abstract void OnStyleUpdate(float styleBonus);
		public virtual void AmyAspect() { Console.WriteLine("AMY ASPECT"); }
		public virtual void KnucklesAspect() { Console.WriteLine("KNUCKLES ASPECT"); }
		public virtual void SonicAspect() { Console.WriteLine("SONIC ASPECT"); }
		public virtual void TailsAspect() { Console.WriteLine("TAILS ASPECT"); }
		public virtual void InitCharacter() { Console.WriteLine("CHARACTER CLASS INIT"); }
		public virtual void UpdateCyloopBehavior()
		{
			if (Utilities.GetBattleInformation("Cyloop") < Scoring.CyloopGauge)
			{
				Utilities.UIToggle("LetterBox", "enable");
				Lua.Call("SetQuickCyloopGauge", 120.0f);
				Scoring.CyloopUses += 1;
				if (Scoring.CyloopUses >= Scoring.CyloopCounter)
				{
					Scoring.CyloopUses = 0;
					Lua.Call("SetQuickCyloopGauge", 0.0f);
				}
				if (SkillLink.ChainActive)
					SkillLink.ComboCall("Cyloop");
			}
			Scoring.CyloopGauge = Utilities.GetBattleInformation("Cyloop");
		}
		public virtual void ResetByDamage()
		{
			Aspects.aspectBonusCrit = 0f;
			Aspects.aspectBonusCritDamage = 0f;
			
			var SonicParams = Reflection.GetDataInfo<SonicParameters.Root>("player_common");
			var AmyParams = Reflection.GetDataInfo<AmyParameters.Root>("amy_common");
			var KnucklesParams = Reflection.GetDataInfo<KnucklesParameters.Root>("knuckles_common");
			var TailsParams = Reflection.GetDataInfo<TailsParameters.Root>("tails_common");
			//I couldn't find a clean way to do this. Time to copy-paste.
			if (currentAspect == "Sonic")
			{
				RFL_SET_PARAM(SonicParams, common.commonPackage.attack.common.criticalDamageRate, 3.25f);
				RFL_SET_PARAM(AmyParams, common.commonPackage.attack.common.criticalDamageRate, 3.25f);
				RFL_SET_PARAM(KnucklesParams, common.commonPackage.attack.common.criticalDamageRate, 3.25f);
				RFL_SET_PARAM(TailsParams, common.commonPackage.attack.common.criticalDamageRate, 3.25f);
			}
			else
			{
				RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.common.criticalDamageRate);
				RFL_RESET_PARAM(AmyParams, AmyParameters.Root, common.commonPackage.attack.common.criticalDamageRate);
				RFL_RESET_PARAM(KnucklesParams, KnucklesParameters.Root, common.commonPackage.attack.common.criticalDamageRate);
				RFL_RESET_PARAM(TailsParams, TailsParameters.Root, common.commonPackage.attack.common.criticalDamageRate);
			}
			RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.common.criticalRate);
			RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.common.offensive.pointMin); 
			RFL_RESET_PARAM(SonicParams, SonicParameters.Root,  common.commonPackage.attack.common.offensive.pointMax); 
			//-------------------------------------------------------------------------------------------------------//
			RFL_RESET_PARAM(AmyParams, AmyParameters.Root, common.commonPackage.attack.common.criticalRate);
			RFL_RESET_PARAM(AmyParams, AmyParameters.Root, common.commonPackage.attack.common.offensive.pointMin); 
			RFL_RESET_PARAM(AmyParams, AmyParameters.Root,  common.commonPackage.attack.common.offensive.pointMax);
			//-------------------------------------------------------------------------------------------------------//
			RFL_RESET_PARAM(KnucklesParams, KnucklesParameters.Root, common.commonPackage.attack.common.criticalRate);
			RFL_RESET_PARAM(KnucklesParams, KnucklesParameters.Root, common.commonPackage.attack.common.offensive.pointMin); 
			RFL_RESET_PARAM(KnucklesParams, KnucklesParameters.Root,  common.commonPackage.attack.common.offensive.pointMax);
			//-------------------------------------------------------------------------------------------------------//
			RFL_RESET_PARAM(TailsParams, TailsParameters.Root, common.commonPackage.attack.common.criticalRate);
			RFL_RESET_PARAM(TailsParams, TailsParameters.Root, common.commonPackage.attack.common.offensive.pointMin); 
			RFL_RESET_PARAM(TailsParams, TailsParameters.Root,  common.commonPackage.attack.common.offensive.pointMax);
			//-------------------------------------------------------------------------------------------------------//
		}
		public static void RestoreAttributes(string aspect)
		{
			//Utilities.UIToggle("Tutorial", "reset");
			var SonicParams = Reflection.GetDataInfo<SonicParameters.Root>("player_common");
			var HitstopParam = Reflection.GetDataInfo<GameHitStopParameter.Root>("hitstop");
			Scoring.aspectScoreMultiplier = 1f;
			if (SonicParams.pData == null)
			{
				Console.WriteLine("INVALID!!!!");
				return;
			}
			var critDamageRate = RFL_GET_PARAM(SonicParams, common.commonPackage.attack.common.criticalRate);
			if (critDamageRate == 3.25f)
			{
				RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.common.criticalDamageRate);
				Aspects.baseCritDamage = 3.0f;
			}
			switch (aspect) //Rewrite this to switch on playerName, not Aspect
			{
				case "Amy": case "Knuckles": case "Sonic": case "Tails":
					Aspects.ReduceDamageMultiplier();
					//RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.common.offensive.pointMin); 
					//RFL_RESET_PARAM(SonicParams, SonicParameters.Root,  common.commonPackage.attack.common.offensive.pointMax); 
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.acceleMode.declineSpeed);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.acceleMode.declineSpeedAccele);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, forwardView.modePackage.jump.gravitySize);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, forwardView.modePackage.parry.justEffectTime2);
					RFL_SET_PARAM(SonicParams, forwardView.modePackage.parry.maxRecieveTimes[0], 0.5f);
					RFL_SET_PARAM(SonicParams, forwardView.modePackage.parry.maxRecieveTimes[1], 0.5f);
					RFL_SET_PARAM(SonicParams, forwardView.modePackage.parry.maxRecieveTimes[2], 0.5f);
					RFL_SET_PARAM(SonicParams, forwardView.modePackage.parry.maxRecieveTimes[3], 0.5f);
					//RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.common.criticalDamageRate);
					//RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.common.criticalRate);
					//RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.cyloop.auraColor.R);
					//RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.cyloop.auraColor.G);
					//RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.cyloop.auraColor.B);
					#region Reset individual move properties
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.stompingAttackSet.sonic.cameraName);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.stompingAttackSet.sonic.riseTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.stompingAttackSet.sonic.motionTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.chargeAtackSet.sonic.riseTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.chargeAtackSet.sonic.lastHitTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.chargeAtackSet.sonic.riseSlowRatio);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.chargeAtackSet.sonic.cameraName);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.loopKickSet.sonic.loopTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.loopKickSet.sonic.cameraName);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.crasherSet.sonic.startWait);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.crasherSet.sonic.zigzagBeginOneStepTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.crasherSet.sonic.zigzagEndOneStepTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.spinSlashSet.sonic.chargeTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.spinSlashSet.sonic.bounceTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.spinSlashSet.sonic.slashTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.spinSlashSet.sonic.lastHitTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.homingShotSet.sonic.appearTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.homingShotSet.sonic.chargeTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.homingShotSet.sonic.spawnTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.homingShotSet.sonic.launchPreWaitTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.homingShotSet.sonic.launchWaitTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.homingShotSet.sonic.cameraName);
					
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.spinSlash.addComboValue);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.spinSlash.addComboValueAccele);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.pursuitKick.damageRate);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.pursuitKick.velocity.Y);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.pursuitKick.velocity.Z);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.sonicBoom.damageRate);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.sonicBoom.attributes);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.sonicBoom.velocity.Y);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.sonicBoom.velocity.Z);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.sonicBoom.velocityKeepTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.crossSlash.velocity.Y);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.crossSlash.attributes);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.crossSlash.velocity.Z);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.crossSlash.velocityKeepTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.crasher.attributes);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.crasher.velocityKeepTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.crasher.velocity.Y);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.crasher.velocity.Z);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.loopKick.attributes);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.loopKick.velocityKeepTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.loopKick.velocity.Y);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.loopKick.velocity.Z);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.aerialAccele1.velocityKeepTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.aerialAccele2.velocityKeepTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.accele1.velocity.Y);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.accele1.velocityKeepTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.accele2.velocity.Y);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.accele2.velocityKeepTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.accele1.attributes);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.accele2.attributes);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.spinSlashLast.velocity.Y);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.spinSlashLast.velocityKeepTime);
					RFL_RESET_PARAM(SonicParams, SonicParameters.Root, common.commonPackage.attack.spinSlashLast.attributes);
					#endregion
					RFL_RESET_PARAM(HitstopParam, GameHitStopParameter.Root, data[20].time);
				break;
			}
		}
		
		public void SetEffectByAspect(string aspect)
		{
			switch (aspect)
			{
				case "Sonic":
					//Utilities.UIToggle("Tutorial", "enable", "Moderately reduces damage in exchange for faster attack speed. Successful Parries increase your Critical Strike chance, tripling the damage dealt.", "Aspect: Sonic");
					Player.Effect.PlayEffect("AspectOn", "ef_so_stomp_end01_third"); //Big blast
					Player.Effect.PlayEffect("AspectOn", "ec_so_skill_airtrick01_hand_cross01_cross01");
					Player.Effect.PlayEffect("AspectOn", "ef_so_object_connect01"); //Orb below the feet
					Player.Effect.PlayEffect("AspectOn", "ec_so_skill_accelerator_aura01_gpu01");
					return;
				case "Amy":
					//Utilities.UIToggle("Tutorial", "enable", "Greatly reduces damage in exchange for lower gravity and stronger aerial combat. Most attacks will launch enemies skyward, and successful Parries will raise your minimum damage.", "Aspect: Amy");
					Player.Effect.PlayEffect("AspectOn", "ec_amp_cyhammer_stomp01_heart01");
					for (int i = 0; i < 13; i++)
						Player.Effect.PlayEffect("AspectOn", "ec_amp_cyhammer_hold01_heart01_gpu01"); //Foot effect
					return;
				case "Knuckles":
					//Utilities.UIToggle("Tutorial", "enable", "Moderately increases knockback and damage dealt, massively increases the drain rate of Phantom Rush. Successful Parries will raise your maximum damage.", "Aspect: Knuckles");
					Player.Effect.PlayEffect("AspectOn", "ef_knp_stomp_bounce01"); //Small burst around the player
					Player.Effect.PlayEffect("AspectOn", "ec_knp_stomp_end01_gpu02"); //Particle burst
					Player.Effect.PlayEffect("AspectOn", "ef_knp_cyknuckle_omen01"); //Fire trail 
					return;
				case "Tails":
					//Utilities.UIToggle("Tutorial", "enable", "Vastly reduces damage dealt and Style gained but halves the drain rate of Phantom Rush. Lengthens the Parry window and regenerates Rings on a successful Parry.", "Aspect: Tails");
					Player.Effect.PlayEffect("AspectOn", "ec_tap_cyclone_appear01_particle01");
					for (int i = 0; i < 5; i++)
						Player.Effect.PlayEffect("AspectOn", "ec_tap_cyclone_appear01_glowring01");
					Player.Effect.PlayEffect("AspectOn", "laser_charge_gpu_foot01");
					Player.Effect.PlayEffect("AspectOn", "laser_charge_gpu01");
					Player.Effect.PlayEffect("AspectOn", "laser_charge_gpu02");
					Player.Effect.PlayEffect("AspectOn", "laser_charge_lightning03");
					Player.Effect.PlayEffect("AspectOn", "line_sphere");
					return;
			}
		}
		public void ToggleAspect()
		{
			if (!hasAspect)
			{
				return;
			}
			if (Player.Input.IsDown(Player.InputActionType.PlayerLightDash))
			{
				Player.State.Discard(Sonic.StateID.StateJump);
				Player.State.Discard(Sonic.StateID.StateHomingAttack);
				if (Player.Input.IsPressed(Player.InputActionType.PlayerJump))
				{
					RestoreAttributes(currentAspect);
					for (int i = 0; i < 15; i++)
						Player.Effect.StopEffect("AspectOn");
					Console.WriteLine("ASPECT ON: AMY");
					if (currentAspect != "Amy")
					{
						SetEffectByAspect("Amy");
						AmyAspect();
					}
					else
						currentAspect = "";
				}
				else if (Player.Input.IsPressed(Player.InputActionType.PlayerAttack))
				{
					RestoreAttributes(currentAspect);
					for (int i = 0; i < 15; i++)
						Player.Effect.StopEffect("AspectOn");
					Console.WriteLine("ASPECT ON: KNUCKLES");
					if (currentAspect != "Knuckles")
					{
						SetEffectByAspect("Knuckles");
						KnucklesAspect();
					}
					else
						currentAspect = "";
				}
				else if (Player.Input.IsPressed(Player.InputActionType.PlayerCyloop))
				{
					RestoreAttributes(currentAspect);
					for (int i = 0; i < 15; i++)
						Player.Effect.StopEffect("AspectOn");
					Console.WriteLine("ASPECT ON: SONIC");
					if (currentAspect != "Sonic")
					{
						SetEffectByAspect("Sonic");
						SonicAspect();
					}
					else
						currentAspect = "";
				}
				else if (Player.Input.IsPressed(Player.InputActionType.PlayerStomping))
				{
					RestoreAttributes(currentAspect);
					for (int i = 0; i < 15; i++)
						Player.Effect.StopEffect("AspectOn");
					Console.WriteLine("ASPECT ON: TAILS");
					if (currentAspect != "Tails")
					{
						SetEffectByAspect("Tails");
						TailsAspect();
					}
					else
						currentAspect = "";
				}
			} 
			else
			{
				Player.State.Restore(Sonic.StateID.StateJump);
				Player.State.Restore(Sonic.StateID.StateHomingAttack);
			}
		}
		public void PlayTaunt()
		{
			if (!hasTaunt)
			{
				Console.WriteLine("Does not have taunt....");
				return;
			}
			TauntBase myTaunt = Taunts[tauntIndex];
			if (myTaunt.useLuaVoice)
				Lua.Call("PlayVoice", myTaunt.tauntVoice);
			else
				Player.Sound.PlaySound(myTaunt.tauntVoice);
			Utilities.DiscardStateByTime(Sonic.StateID.StateSquat, myTaunt.tauntDuration);
			Console.WriteLine("ANIM: " + myTaunt.tauntAnim);
			Player.Animation.SetAnimation(myTaunt.tauntAnim);
			tauntIndex += 1;
			if (tauntIndex > Taunts.Count - 1)
				tauntIndex = 0;
		}		
	}
	#region Character Class Declarations
	public class Sonic_CDX : CharacterBase
	{
		#region MoveBonuses
		public List<SkillBonus> Parry = new()
		{
			new SkillBonus("Duration", "Rush", 1.3f, 3.3f),
			new SkillBonus("Duration", "Style", 1.3f, 3.3f),
			new SkillBonus("Unlimited", "Rush", 1.3f, 3.3f)
		}
		public List<SkillBonus> SonicBoom = new()
		{
			new SkillBonus("Duration", "Rush", 1.1f, 3.0f),
			new SkillBonus("Duration", "Style", 1.15f, 6.0f),
			new SkillBonus("Duration", "Cyloop", 1.3f, 12.0f)
		}
		public List<SkillBonus> CrossSlash = new()
		{
			new SkillBonus("Duration", "Cyloop", 1.2f, 3.0f),
			new SkillBonus("Duration", "Style", 1.25f, 5.0f),
			new SkillBonus("Duration", "StyleRush", 1.4f, 10.0f)
		}
		public List<SkillBonus> LoopKick = new()
		{
			new SkillBonus("Single", "Rush", 1.5f, 2.5f),
			new SkillBonus("Single", "Style", 2.0f, 3.5f),
			new SkillBonus("Single", "StyleRush", 2.5f, 5.0f)
		}
		public List<SkillBonus> HomingShot = new()
		{
			new SkillBonus("Duration", "Rush", 1.3f, 4.0f),
			new SkillBonus("Duration", "Cyloop", 1.4f, 10.0f),
			new SkillBonus("Pause", "Rush", 2.0f, 15.0f)
		}
		public List<SkillBonus> Stomp = new()
		{
			new SkillBonus("Single", "Cyloop", 1.15f, 4.0f),
			new SkillBonus("Single", "Style", 1.25f, 6.0f),
			new SkillBonus("Single", "StyleRush", 1.4f, 9.0f)
		}
		public List<SkillBonus> CycloneKick = new()
		{
			new SkillBonus("Duration", "Cyloop", 1.1f, 2.66f),
			new SkillBonus("Duration", "Style", 1.3f, 4.0f),
			new SkillBonus("Duration", "StyleRush", 1.5f, 6.5f)
		}
		public List<SkillBonus> Crasher = new()
		{
			new SkillBonus("Single", "Cyloop", 1.15f, 5.0f),
			new SkillBonus("Single", "Rush", 1.2f, 5.5f),
			new SkillBonus("Single", "StyleRush", 2.5f, 10.5f)
		}
		public List<SkillBonus> Smash = new()
		{
			new SkillBonus("Single", "StyleRush", 1.5f, 5.0f),
			new SkillBonus("Duration", "StyleRush", 2.0f, 7.5f),
			new SkillBonus("Unlimited", "Rush", 3.0f, 12.5f)
		}
		public List<SkillBonus> Standard = new()
		{
			new SkillBonus("Item", "Ring", 1.1f, 2.0f),
			new SkillBonus("Item", "Ring", 1.3f, 2.5f),
			new SkillBonus("Item", "Ring", 2.5f, 4.0f)
		}
		public List<SkillBonus> Cyloop = new()
		{
			new SkillBonus("Duration", "CriticalRate", 1.35f, 10.0f),
			new SkillBonus("Duration", "CriticalDamage", 1.55f, 10.0f),
			new SkillBonus("Duration", "CriticalAll", 2.0f, 10.0f)
		}
		#endregion
		public Sonic_CDX()
		{
			hasTaunt = true;
			hasAspect = true;
			hasSkillLink = true;
			Moves = new() //Establishes Style bonuses for each move
			{
				new MoveBase(20,25,30,35,40), //Parry  0
				new MoveBase(10,7,5,4,3), //Sonic Boom 1
				new MoveBase(15, 10, 8, 5, 4), //Cross Slash 2
				new MoveBase(30,20,10,5,5), //Loop Kick 3
				new MoveBase(15,13,11,10,9), //Homing Shot 4
				new MoveBase(15,10,5,3,1), //Stomp 5
				new MoveBase(25,20,15,10,5), //Cyclone Kick ("CHARGE") 6
				new MoveBase(20,15,10,7,5), //Wild Rush ("Crasher") 7
				new MoveBase(20,10,5,3,1), //Grand Slam 8
				new MoveBase(6,5,4,3,3), //Punch/kick 9
				new MoveBase(10,8,6,5,4), //Slide 10
				new MoveBase(25,25,25,25,25), //Finisher Neutral, impossible to reach other values   11
				new MoveBase(25,25,25,25,25), //Finisher Forward   StateHomingFinish (StateHomingFinishED is used for homing attack)  12
				new MoveBase(17,17,17,17,17), //Finisher Back  13
				new MoveBase(30,30,30,30,30), //Finisher Left  14
				new MoveBase(30,30,30,30,30), //Finisher Right  15
				new MoveBase(30,10,5,5,5), //Recovery Smash  16
				new MoveBase(15,10,7,4,1), //Pursuit Kick  17
				new MoveBase(20,40,60,80,100) //Quick Cyloop  18
			};
			Taunts = new() //Anim||voice line||anim duration
			{
				new TauntBase("SEND_SIGNAL", "ev2080_030", 1.0f),
				new TauntBase("AWAKENING", "si4000_032", 1.3f),
				//new TauntBase("IDLE_ACT08_SV", "", 1.5f),
				//new TauntBase("IDLE_ACT09_SV", "", 1.5f)
			};
			MoveBonuses = new()
			{
				{"Parry", Parry},
				{"SonicBoom", SonicBoom},
				{"CrossSlash", CrossSlash},
				{"LoopKick", LoopKick},
				{"HomingShot", HomingShot},
				{"Stomp", Stomp},
				{"CycloneKick", CycloneKick},
				{"Crasher", Crasher},
				{"Smash", Smash},
				{"Standard", Standard},
				{"Cyloop", Cyloop}
			};
			myRanks = new()
			{
				new RankLevel("Super Sonic!!!", 500, 6, "si2000_037"),
				new RankLevel("S", 350, 5, "si2000_024"),
				new RankLevel("A", 220, 4, "si2000_023"),
				new RankLevel("B", 140, 3, "si2000_022"),
				new RankLevel("C", 80, 2, "si2000_021"),
				new RankLevel("D", 40, 1, "si2000_020", 0)
			};
			soundList = new()
			{
				{ "Amy", new List<string>{"aa1101", "qu1500_050", "ev1110_070"} },
				{ "Knuckles", new List<string>{"ka1112", "ka1110"} },
				{ "Tails", new List<string>{"ta1112", "ta1110", "sa1036"} },
				{ "Sonic", new List<string>{"sa1051", "si3000_023", "si4000_031"} }
			};
			
		}
		public override void SonicAspect()
		{
			//ef_so_absorb_range01 yellowish ring around Sonic
			currentAspect = "Sonic";
			Scoring.aspectScoreMultiplier = 0.5f;
			Aspects.baseCritDamage = 3.25f;
			int slot = rnd.Next(0, soundList[currentAspect].Count);
			Player.Sound.PlaySound("sn_skateboard_change");
			if (slot == 0)
				Player.Sound.PlaySound(soundList[currentAspect][slot]);
			else
				Lua.Call("PlayVoice", soundList[currentAspect][slot]);
			var SonicParams = Reflection.GetDataInfo<SonicParameters.Root>("player_common");
			var HitstopParam = Reflection.GetDataInfo<GameHitStopParameter.Root>("hitstop");
			var critDamageRate = RFL_GET_PARAM(SonicParams, common.commonPackage.attack.common.criticalDamageRate);
			if (critDamageRate == 3f)
				RFL_SET_PARAM(SonicParams, common.commonPackage.attack.common.criticalDamageRate, 3.25f);
			#region Speed up moves
			RFL_SET_PARAM(SonicParams, common.chargeAtackSet.sonic.riseTime, 0.75f);
			RFL_SET_PARAM(SonicParams, common.chargeAtackSet.sonic.riseSlowRatio, 0.2f);
			RFL_SET_PARAM(SonicParams, common.chargeAtackSet.sonic.lastHitTime, 0.65f);
			RFL_SET_PARAM(SonicParams, common.chargeAtackSet.sonic.cameraName, "Spinslash");
			
			RFL_SET_PARAM(SonicParams, common.loopKickSet.sonic.loopTime, 0.5f);
			RFL_SET_PARAM(SonicParams, common.loopKickSet.sonic.cameraName, "(null)");
			
			RFL_SET_PARAM(SonicParams, common.crasherSet.sonic.startWait, 0.15f);
			RFL_SET_PARAM(SonicParams, common.crasherSet.sonic.zigzagBeginOneStepTime, 0.07f);
			RFL_SET_PARAM(SonicParams, common.crasherSet.sonic.zigzagEndOneStepTime, 0.07f);
			
			RFL_SET_PARAM(SonicParams, common.spinSlashSet.sonic.chargeTime, 0.2f);
			RFL_SET_PARAM(SonicParams, common.spinSlashSet.sonic.bounceTime, 0.35f);
			RFL_SET_PARAM(SonicParams, common.spinSlashSet.sonic.slashTime, 1.25f);
			RFL_SET_PARAM(SonicParams, common.spinSlashSet.sonic.lastHitTime, 0.875f);
			
			RFL_SET_PARAM(SonicParams, common.homingShotSet.sonic.appearTime, 0.25f);
			RFL_SET_PARAM(SonicParams, common.homingShotSet.sonic.chargeTime, 0.55f);
			RFL_SET_PARAM(SonicParams, common.homingShotSet.sonic.spawnTime, 0.2f);
			RFL_SET_PARAM(SonicParams, common.homingShotSet.sonic.launchPreWaitTime, 0.25f);
			RFL_SET_PARAM(SonicParams, common.homingShotSet.sonic.cameraName, "(null)");
			
			RFL_SET_PARAM(SonicParams, common.stompingAttackSet.sonic.riseTime, 0.15f);
			RFL_SET_PARAM(SonicParams, common.stompingAttackSet.sonic.motionTime, 0.5f);
			RFL_SET_PARAM(SonicParams, common.stompingAttackSet.sonic.cameraName, "SpinslashAfter");
			#endregion
			RFL_SET_PARAM(SonicParams, forwardView.modePackage.parry.justEffectTime2, 1.5f);
			RFL_SET_PARAM(HitstopParam, data[20].time, 1.5f);
			Aspects.ReduceDamageMultiplier(0.825f);
			Aspects.SetRushDrain(9f, 25f);
		}
		public override void AmyAspect()
		{
			currentAspect = "Amy";
			int slot = rnd.Next(0,soundList[currentAspect].Count);
			Player.Sound.PlaySound("sn_skateboard_change");
			if (slot == 0)
				Player.Sound.PlaySound(soundList[currentAspect][slot]);
			else
				Lua.Call("PlayVoice", soundList[currentAspect][slot]);
			var SonicParams = Reflection.GetDataInfo<SonicParameters.Root>("player_common");
			var HitstopParam = Reflection.GetDataInfo<GameHitStopParameter.Root>("hitstop");
			//FASTER PHANTOM RUSH DEPLETION
			/*RFL_SET_PARAM(SonicParams, common.cyloop.auraColor.R, 255f/255f);
			RFL_SET_PARAM(SonicParams, common.cyloop.auraColor.G, 192f/255f);
			RFL_SET_PARAM(SonicParams, common.cyloop.auraColor.B, 203f/255f);*/
			Aspects.SetRushDrain(7f, 22.5f);
			#region EnableMoveJuggling
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.pursuitKick.velocity.Y, 13.25f);
			
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.sonicBoom.velocity.Y, 0.25f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.sonicBoom.velocityKeepTime, 2.5f);
			
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crossSlash.velocity.Z, -8f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crossSlash.velocityKeepTime, 0.33f);
			
			
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crasher.attributes, 49184);
			//RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crasher.velocityKeepTime, 0.15f);
			//RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crasher.velocity.Y, 7.5f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crasher.velocityKeepTime, 0.35f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crasher.velocity.Y, 13.5f);
			
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.loopKick.attributes, 49184);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.loopKick.velocity.Z, 13f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.loopKick.velocity.Y, 10f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.loopKick.velocityKeepTime, 0.5f);
			
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.aerialAccele1.velocityKeepTime, 1.7f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.aerialAccele2.velocityKeepTime, 1.7f);
			
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.common.offensive.pointMin, 1);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.common.offensive.pointMax, 5);
			
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.accele1.attributes, 49184);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.accele2.attributes, 49184);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.accele1.velocity.Y, 10.25f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.accele1.velocityKeepTime, 0.15f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.accele2.velocity.Y, 10.25f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.accele2.velocityKeepTime, 0.15f);
			
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.spinSlashLast.attributes, 49536);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.spinSlashLast.velocity.Y, 0.33f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.spinSlashLast.velocityKeepTime, 7.33f);
			#endregion
			RFL_SET_PARAM(SonicParams, forwardView.modePackage.jump.gravitySize, 15f);
			RFL_SET_PARAM(SonicParams, forwardView.modePackage.parry.justEffectTime2, 0.5f);
			RFL_SET_PARAM(HitstopParam, data[20].time, 0.5f);
			//Aspects.ReduceDamageMultiplier(0.65f);
		}
		public override void TailsAspect()
		{
			Scoring.aspectScoreMultiplier = 0.5f;
			currentAspect = "Tails";
			//Player.Effect.PlayEffect("AspectOn", "ef_tap_pilebunker_end01"); Too flashy
			int slot = rnd.Next(0, soundList[currentAspect].Count);
			Player.Sound.PlaySound("sn_skateboard_change");
			Player.Sound.PlaySound(soundList[currentAspect][slot]);
			var SonicParams = Reflection.GetDataInfo<SonicParameters.Root>("player_common");
			var HitstopParam = Reflection.GetDataInfo<GameHitStopParameter.Root>("hitstop");
			Aspects.SetRushDrain(3.5f, 10f);
			Aspects.ReduceDamageMultiplier(0.05f);
			#region Set move effects
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.spinSlash.addComboValue, -0.15f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.spinSlash.addComboValueAccele, -0.15f);
			#endregion
			RFL_SET_PARAM(SonicParams, forwardView.modePackage.parry.maxRecieveTimes[0], 1f);
			RFL_SET_PARAM(SonicParams, forwardView.modePackage.parry.maxRecieveTimes[1], 1f);
			RFL_SET_PARAM(SonicParams, forwardView.modePackage.parry.maxRecieveTimes[2], 1f);
			RFL_SET_PARAM(SonicParams, forwardView.modePackage.parry.maxRecieveTimes[3], 1f);
		}
		public override void KnucklesAspect()
		{
			currentAspect = "Knuckles";
			int slot = rnd.Next(0,soundList[currentAspect].Count);
			Player.Sound.PlaySound("sn_skateboard_change");
			Player.Sound.PlaySound(soundList[currentAspect][slot]);
			//Player.Effect.PlayEffect("AspectOn", "ef_knp_stomp_end01"); Big ring of fire, very flashy
			var SonicParams = Reflection.GetDataInfo<SonicParameters.Root>("player_common");
			var HitstopParam = Reflection.GetDataInfo<GameHitStopParameter.Root>("hitstop");
			Aspects.ReduceDamageMultiplier(1.35f);
			RFL_SET_PARAM(SonicParams, forwardView.modePackage.jump.gravitySize, 35f);
			RFL_SET_PARAM(SonicParams, forwardView.modePackage.parry.justEffectTime2, 2.5f);
			RFL_SET_PARAM(HitstopParam, data[20].time, 2.5f);
			Aspects.SetRushDrain(12f, 35f);
			#region EnableMoveSpikes
			//RFL_SET_PARAM(SonicParams, common.commonPackage.attack.pursuitKick.damageRate, 1.5f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.pursuitKick.velocity.Y, -13.25f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.pursuitKick.velocity.Z, 13.25f);
			
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.loopKick.attributes, 49184);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.loopKick.velocity.Z, 15f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.loopKick.velocity.Y, -16f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.loopKick.velocityKeepTime, 1.0f);
			
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.spinSlashLast.attributes, 200);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.spinSlashLast.velocity.Y, -30f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.spinSlashLast.velocityKeepTime, 0.33f);
			
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crasher.attributes, 49184);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crasher.velocityKeepTime, 0.35f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crasher.velocity.Z, 26.5f);
			
			
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.sonicBoom.velocity.Z, 4.25f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.sonicBoom.velocityKeepTime, 0.5f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.sonicBoom.damageRate, 0.5f);
			
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crossSlash.attributes, 49184);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crossSlash.velocity.Y, 10f);
			RFL_SET_PARAM(SonicParams, common.commonPackage.attack.crossSlash.velocityKeepTime, 0.73f);
			#endregion
		}
		public override void OnStyleUpdate(float styleBonus)
		{
			float prGauge = Utilities.GetBattleInformation("PhantomRush");
			float prBonus = prGauge + styleBonus * Scoring.rushMultiplier * Scoring.single_RushMultiplier;
			float qclGauge = Utilities.GetBattleInformation("Cyloop");
			float qclBonus = qclGauge + styleBonus * Scoring.CyloopMultiplier;
			float addStyle = styleBonus * Scoring.scoreMultiplier * Scoring.single_ScoreMultiplier * Scoring.difficultyMultiplier;
			Scoring.styleScore += (int)(addStyle * Scoring.aspectScoreMultiplier);
			if (Scoring.styleScore > Scoring.maxStyleScore)
				Scoring.maxStyleScore = Scoring.styleScore;
			Lua.Call("SetPhantomRushGauge", prBonus);
			Lua.Call("SetQuickCyloopGauge", Math.Min(120.0f, qclBonus));
			Scoring.StyleRankCall(myRanks);
		}
	}
	public class Super_CDX : CharacterBase
	{
		public int[] MeterDrain = new int[]
		{
			0,  //Parry
			5,  //SonicBoom
			15, //Cross Slash
			30, //Loop Kick
			35, //Homing Shot
			15, //Stomp
			15, //Cyclone Kick
			14, //Crasher
			35, //Grand Slam
			3,  //Punch
			3,  //Kick
			7,  //Finisher Neutral
			7,  //Finisher Forward
			7,  //Finisher Back
			7,  //Finisher Left
			7,  //Finisher Right
			35, //Recovery Smash
			15, //Pursuit Kick
			0   //Quick Cyloop
		};
		private Dictionary<string, int> ringStart = new()
		{
			{ "Easy", 		300 },
			{ "Normal", 	120 },
			{ "Hard", 		60 	},
			{ "Extreme",	30  }
		}
		#region MoveBonuses
		public List<SkillBonus> Parry = new()
		{
			new SkillBonus("Duration", "Rush", 1.3f, 3.3f),
			new SkillBonus("Duration", "Style", 1.3f, 3.3f),
			new SkillBonus("Unlimited", "Rush", 1.3f, 3.3f)
		}
		public List<SkillBonus> SonicBoom = new()
		{
			new SkillBonus("Duration", "Rush", 1.1f, 3.0f),
			new SkillBonus("Duration", "Style", 1.15f, 6.0f),
			new SkillBonus("Duration", "Cyloop", 1.3f, 12.0f)
		}
		public List<SkillBonus> CrossSlash = new()
		{
			new SkillBonus("Duration", "Cyloop", 1.2f, 3.0f),
			new SkillBonus("Duration", "Style", 1.25f, 5.0f),
			new SkillBonus("Duration", "StyleRush", 1.4f, 10.0f)
		}
		public List<SkillBonus> LoopKick = new()
		{
			new SkillBonus("Single", "Rush", 1.5f, 2.5f),
			new SkillBonus("Single", "Style", 2.0f, 3.5f),
			new SkillBonus("Single", "StyleRush", 2.5f, 5.0f)
		}
		public List<SkillBonus> HomingShot = new()
		{
			new SkillBonus("Duration", "Rush", 1.3f, 4.0f),
			new SkillBonus("Duration", "Cyloop", 1.4f, 10.0f),
			new SkillBonus("Pause", "Rush", 2.0f, 15.0f)
		}
		public List<SkillBonus> Stomp = new()
		{
			new SkillBonus("Single", "Cyloop", 1.15f, 4.0f),
			new SkillBonus("Single", "Style", 1.25f, 6.0f),
			new SkillBonus("Single", "StyleRush", 1.4f, 9.0f)
		}
		public List<SkillBonus> CycloneKick = new()
		{
			new SkillBonus("Duration", "Cyloop", 1.1f, 2.66f),
			new SkillBonus("Duration", "Style", 1.3f, 4.0f),
			new SkillBonus("Duration", "StyleRush", 1.5f, 6.5f)
		}
		public List<SkillBonus> Crasher = new()
		{
			new SkillBonus("Single", "Cyloop", 1.15f, 5.0f),
			new SkillBonus("Single", "Rush", 1.2f, 5.5f),
			new SkillBonus("Single", "StyleRush", 2.5f, 10.5f)
		}
		public List<SkillBonus> Smash = new()
		{
			new SkillBonus("Single", "StyleRush", 1.5f, 5.0f),
			new SkillBonus("Duration", "StyleRush", 2.0f, 7.5f),
			new SkillBonus("Unlimited", "Rush", 3.0f, 12.5f)
		}
		public List<SkillBonus> Standard = new()
		{
			new SkillBonus("Item", "Ring", 1.1f, 2.0f),
			new SkillBonus("Item", "Ring", 1.3f, 2.5f),
			new SkillBonus("Item", "Ring", 2.5f, 4.0f)
		}
		public List<SkillBonus> Cyloop = new()
		{
			new SkillBonus("Duration", "CriticalDamage", 1.35f, 10.0f),
			new SkillBonus("Duration", "CriticalDamage", 1.55f, 10.0f),
			new SkillBonus("Duration", "CriticalDamage", 2.0f, 10.0f)
		}
		#endregion
		public Super_CDX()
		{
			hasSetRush = false;
			hasTaunt = false;
			hasAspect = false;
			hasSkillLink = true;
			MoveBonuses = new()
			{
				{"Parry", Parry},
				{"SonicBoom", SonicBoom},
				{"CrossSlash", CrossSlash},
				{"LoopKick", LoopKick},
				{"HomingShot", HomingShot},
				{"Stomp", Stomp},
				{"CycloneKick", CycloneKick},
				{"Crasher", Crasher},
				{"Smash", Smash},
				{"Standard", Standard},
				{"Cyloop", Cyloop}
			};
			myRanks = new()
			{
				new RankLevel("Super Sonic!!!", 750, 6, "si2000_037"),
				new RankLevel("S", 500, 5, "si2000_024"),
				new RankLevel("A", 350, 4, "si2000_023"),
				new RankLevel("B", 220, 3, "si2000_022"),
				new RankLevel("C", 140, 2, "si2000_021"),
				new RankLevel("D", 0, 1, "si2000_020", 0)
			};
			Moves = new() //Establishes Style bonuses for each move
			{
				new MoveBase(30,35,40,45,50), //Parry
				new MoveBase(25,15,7,4,3), //Sonic Boom A
				new MoveBase(30, 25, 8, 5, 4), //Cross Slash A
				new MoveBase(35,25,10,5,5), //Loop Kick A
				new MoveBase(50,40,25,10,9), //Homing Shot A
				new MoveBase(20,15,10,3,1), //Stomp A
				new MoveBase(25,20,15,10,5), //Cyclone Kick ("CHARGE") A
				new MoveBase(30,20,10,7,5), //Wild Rush ("Crasher") A
				new MoveBase(70,50,30,20,1), //Grand Slam A
				new MoveBase(7,6,5,4,3), //Punch/kick A
				new MoveBase(10,8,6,5,4), //Slide A
				new MoveBase(28,25,25,25,25), //Finisher Neutral, impossible to reach other values
				new MoveBase(25,25,25,25,25), //Finisher Forward   StateHomingFinish (StateHomingFinishED is used for homing attack)
				new MoveBase(20,17,17,17,17), //Finisher Back
				new MoveBase(32,30,30,30,30), //Finisher Left
				new MoveBase(32,30,30,30,30), //Finisher Right
				new MoveBase(30,10,5,5,5), //Recovery Smash
				new MoveBase(25,20,13,12,11), //Pursuit Kick A
				new MoveBase(30,40,60,80,100) //Quick Cyloop A
			};
			warningList = new()
			{
				{"ringWarning", false},
				{"rushWarning", false}
			};
		}
		public override void InitCharacter()
		{
			Console.WriteLine("___INIT SUPER");
			hasInitChar = true;
			warningList["ringWarning"] = false;
			warningList["rushWarning"] = false;
			Aspects.SetRushDrain(0f, 45f);
			var pBlackboardItem = BlackboardItem.Get();
			if (pBlackboardItem == null)
			{
				Console.WriteLine("Check was null, failsafe and return");
				failSafeInitTimer += 0.75f;
				return;
			}
			pBlackboardItem->RingCount = ringStart[Scoring.currentDifficulty];
			Lua.Call("FadeIn", 0.25f);
			
		}
		public override void OnStyleUpdate(float styleBonus)
		{
			int moveNum = MoveIndex[currentAnim];
			//float prGauge = Utilities.GetBattleInformation("PhantomRush");
			//float prBonus = prGauge + styleBonus * Scoring.rushMultiplier * Scoring.single_RushMultiplier;
			float qclGauge = Utilities.GetBattleInformation("Cyloop");
			float qclBonus = qclGauge + styleBonus * Scoring.CyloopMultiplier;
			float addStyle = styleBonus * Scoring.scoreMultiplier * Scoring.single_ScoreMultiplier * Scoring.difficultyMultiplier;
			Scoring.styleScore += (int)(addStyle * Scoring.aspectScoreMultiplier);
			if (Scoring.styleScore > Scoring.maxStyleScore)
				Scoring.maxStyleScore = Scoring.styleScore;
			//Lua.Call("SetPhantomRushGauge", prBonus);
			Lua.Call("SetQuickCyloopGauge", Math.Min(120.0f, qclBonus));
			if (isRush)
			{
				Bonuses.pauseVal -= MeterDrain[moveNum];
			}
			Scoring.StyleRankCall(myRanks);
		}
	}
	public class Amy_CDX : CharacterBase
	{	
		public Amy_CDX()
		{
			hasTaunt = true;
			hasAspect = false; //Temporary. Aspects will be enabled later.
			hasSkillLink = false;
			Moves = new()
			{
				new MoveBase(20, 25, 30, 35, 40), //Parry					0
				new MoveBase(15, 13, 14, 12, 11), //ATK_TAROT 				1
				new MoveBase(17, 14, 11,  9,  8), //ATK_TAROT02 			2
				new MoveBase(18, 15, 13, 12, 10), //TAROT_ROLL & RING_MAX	3
				new MoveBase(30, 20, 10, 10, 10), //THROW_KISS				4
				new MoveBase(15, 10,  5,  3,  1), //STOMPING				5
				new MoveBase(20, 40, 60, 80, 100), //Cyloop					6
			}
			Taunts = new()
			{
				new TauntBase("THROW_KISS", "aa1200", 1.5f, false),
				new TauntBase("PIYORI_START", "sa1048", 1.5f, false)
			}
			MoveBonuses = new() //Disabled for now
			{}
			myRanks = new()
			{
				new RankLevel("Super Sonic!!!", 350, 6, "si2000_037"),
				new RankLevel("S", 290, 5, "si2000_024"),
				new RankLevel("A", 145, 4, "si2000_023"),
				new RankLevel("B", 90, 3, "si2000_022"),
				new RankLevel("C", 65, 2, "si2000_021"),
				new RankLevel("D", 40, 1, "si2000_020", 0)
			}
			soundList = new()
			{
				{ "Amy", new List<string>{"aa1101", "qu1500_050", "ev1110_070"} },
				{ "Knuckles", new List<string>{"ka1112", "ka1110"} },
				{ "Tails", new List<string>{"ta1112", "ta1110", "sa1036"} },
				{ "Sonic", new List<string>{"sa1051", "si3000_023", "si4000_031"} }
			}
		}
		public override void OnStyleUpdate(float styleBonus)
		{
			float qclGauge = Utilities.GetBattleInformation("Cyloop");
			float qclBonus = qclGauge + styleBonus * Scoring.CyloopMultiplier;
			float addStyle = styleBonus * Scoring.scoreMultiplier * Scoring.single_ScoreMultiplier * Scoring.difficultyMultiplier;
			Scoring.styleScore += (int)(addStyle * Scoring.aspectScoreMultiplier);
			if (Scoring.styleScore > Scoring.maxStyleScore)
				Scoring.maxStyleScore = Scoring.styleScore;
			Lua.Call("SetQuickCyloopGauge", Math.Min(120.0f, qclBonus));
			Scoring.StyleRankCall(myRanks);
		}
		public override void UpdateCyloopBehavior()
		{
			if (Utilities.GetBattleInformation("Cyloop") < Scoring.CyloopGauge)
			{
				Utilities.UIToggle("LetterBox", "enable");
				if (SkillLink.ChainActive)
					SkillLink.ComboCall("Cyloop");
			}
			Scoring.CyloopGauge = Utilities.GetBattleInformation("Cyloop");
		}
	}
	public class Knuckles_CDX : CharacterBase
	{	
		public Knuckles_CDX()
		{
			hasTaunt = true;
			hasAspect = false; //Temporary. Aspects will be enabled later.
			hasSkillLink = false;
			Moves = new()
			{
				new MoveBase(20, 25, 30, 35, 40), //Parry					0
				new MoveBase(15, 13, 14, 12, 11), //PUNCH01 				1
				new MoveBase(17, 14, 11,  9,  8), //PUNCH02 				2
				new MoveBase(18, 15, 13, 12, 10), //UPPERCUT				3
				new MoveBase(60, 40, 30, 20, 10), //HEAT_KNUCKLE			4
				new MoveBase(15, 12, 10,  7,  9), //STOMPING				5
				new MoveBase(20, 40, 60, 80, 100), //Cyloop					6
			}
			Taunts = new()
			{
				new TauntBase("HOLEDIVE_END_CLIMB", "ka1110", 1.5f, false),
				new TauntBase("HEAT_KNUCKLE_BOUNCE", "sa1002", 1.5f, false)
			}
			MoveBonuses = new() //Disabled for now
			{}
			myRanks = new()
			{
				new RankLevel("Super Sonic!!!", 350, 6, "si2000_037"),
				new RankLevel("S", 290, 5, "si2000_024"),
				new RankLevel("A", 145, 4, "si2000_023"),
				new RankLevel("B", 90, 3, "si2000_022"),
				new RankLevel("C", 65, 2, "si2000_021"),
				new RankLevel("D", 40, 1, "si2000_020", 0)
			}
			soundList = new()
			{
				{ "Amy", new List<string>{"aa1101", "qu1500_050", "ev1110_070"} },
				{ "Knuckles", new List<string>{"ka1112", "ka1110"} },
				{ "Tails", new List<string>{"ta1112", "ta1110", "sa1036"} },
				{ "Sonic", new List<string>{"sa1051", "si3000_023", "si4000_031"} }
			}
		}
		public override void OnStyleUpdate(float styleBonus)
		{
			float qclGauge = Utilities.GetBattleInformation("Cyloop");
			float qclBonus = qclGauge + styleBonus * Scoring.CyloopMultiplier;
			float addStyle = styleBonus * Scoring.scoreMultiplier * Scoring.single_ScoreMultiplier * Scoring.difficultyMultiplier;
			Scoring.styleScore += (int)(addStyle * Scoring.aspectScoreMultiplier);
			if (Scoring.styleScore > Scoring.maxStyleScore)
				Scoring.maxStyleScore = Scoring.styleScore;
			Lua.Call("SetQuickCyloopGauge", Math.Min(120.0f, qclBonus));
			Scoring.StyleRankCall(myRanks);
		}
		public override void UpdateCyloopBehavior()
		{
			if (Utilities.GetBattleInformation("Cyloop") < Scoring.CyloopGauge)
			{
				Utilities.UIToggle("LetterBox", "enable");
				if (SkillLink.ChainActive)
					SkillLink.ComboCall("Cyloop");
			}
			Scoring.CyloopGauge = Utilities.GetBattleInformation("Cyloop");
		}
	}
	public class Tails_CDX : CharacterBase
	{	
		public Tails_CDX()
		{
			hasTaunt = true;
			hasAspect = false; //Temporary. Aspects will be enabled later.
			hasSkillLink = false;
			Moves = new()
			{
				new MoveBase(20, 25, 30, 35, 40), //Parry					0
				new MoveBase( 8,  8,  6,  4,  2), //Ground Spanner 			1
				new MoveBase(12, 12,  8,  8,  4), //Air Spanner				2
				new MoveBase(18, 15, 13, 12, 10), //Float Spanner			3
				new MoveBase(60, 40, 30, 20, 10), //Cyloop					4
				new MoveBase(15, 12, 10,  7,  9)  //Stomping				5
			}
			Taunts = new()
			{
				new TauntBase("TAILS_BOOST_SWIM", "", 1.5f),
				new TauntBase("CYBLASTER_LIFT_LOOP", "", 1.5f),
			}
			MoveBonuses = new() //Disabled for now
			{}
			myRanks = new()
			{
				new RankLevel("Super Sonic!!!", 350, 6, "si2000_037"),
				new RankLevel("S", 290, 5, "si2000_024"),
				new RankLevel("A", 145, 4, "si2000_023"),
				new RankLevel("B", 90, 3, "si2000_022"),
				new RankLevel("C", 65, 2, "si2000_021"),
				new RankLevel("D", 40, 1, "si2000_020", 0)
			}
			soundList = new()
			{
				{ "Amy", new List<string>{"aa1101", "qu1500_050", "ev1110_070"} },
				{ "Knuckles", new List<string>{"ka1112", "ka1110"} },
				{ "Tails", new List<string>{"ta1112", "ta1110", "sa1036"} },
				{ "Sonic", new List<string>{"sa1051", "si3000_023", "si4000_031"} }
			}
		}
		public override void OnStyleUpdate(float styleBonus)
		{
			float qclGauge = Utilities.GetBattleInformation("Cyloop");
			float qclBonus = qclGauge + styleBonus * Scoring.CyloopMultiplier;
			float addStyle = styleBonus * Scoring.scoreMultiplier * Scoring.single_ScoreMultiplier * Scoring.difficultyMultiplier;
			Scoring.styleScore += (int)(addStyle * Scoring.aspectScoreMultiplier);
			if (Scoring.styleScore > Scoring.maxStyleScore)
				Scoring.maxStyleScore = Scoring.styleScore;
			Lua.Call("SetQuickCyloopGauge", Math.Min(120.0f, qclBonus));
			Scoring.StyleRankCall(myRanks);
		}
		public override void UpdateCyloopBehavior()
		{
			if (Utilities.GetBattleInformation("Cyloop") < Scoring.CyloopGauge)
			{
				Utilities.UIToggle("LetterBox", "enable");
				if (SkillLink.ChainActive)
					SkillLink.ComboCall("Cyloop");
			}
			Scoring.CyloopGauge = Utilities.GetBattleInformation("Cyloop");
		}
	}
	#endregion
	public static void AssignCharacter(dynamic charType)
	{
		if (charType == null)
			return;
		lastPlayerName = playerName
		if (BlackboardStatus.IsSuper())
			playerName = "Super";
		else if (charType == Player.PlayerType.Sonic)
			playerName = "Sonic";
		else if (charType == Player.PlayerType.Amy)
			playerName = "Amy";
		else if (charType == Player.PlayerType.Knuckles)
			playerName = "Knuckles";
		else if (charType == Player.PlayerType.Tails)
			playerName = "Tails";
		if (lastPlayerName != playerName)
		{
			Console.WriteLine("FORCE INIT VIA NAME CHANGE!!!");
			CharacterList[playerName].InitCharacter();
		}
	};
	public static void incrementTaunt(dynamic curChar)
	{
		if (Lua.Call<string>("GetCurrentAnimationName") == "SQUAT_LOOP")
		{
			if (Lua.Call<int>("GetPlayerStatus", "OnGround") == 1)
				tauntTimer += 1f;
			if (tauntTimer >= 20f)
			{
				tauntTimer = 0f;
				curChar.PlayTaunt();
			}
		} else
			tauntTimer = 0f;
	}
	public static Dictionary<string, CharacterBase> CharacterList = new()
	{
		{    "Sonic", new Sonic_CDX() 		},
		{    "Super", new Super_CDX() 		},
		{    "Amy",   new Amy_CDX()   		},
		{    "Knuckles", new Knuckles_CDX() },
		{	 "Tails", new Tails_CDX()		}
	};
	static bool useRankUI;
	static bool useRankUI_Super;
	static bool useLetterbox;
	static bool useLetterbox_Super;
	static bool useAnnouncer;
	static bool useAnnouncer_Super;
	
	Random qteRNG = new Random();
	static string playerName = "Sonic";
	static string lastPlayerName = "Sonic";
	static string currentAnim = "STAND"; //Forces an init
	static string lastAnim = "DEFAULT"; //Not particularly important but may as well give it a value.
	static bool hasChangedRush = false; //Hacky way to solve Cyberspace/Arcade crashes
	static bool isStateDiscarded = false;
	static bool hasInit = true;
	static bool isRankUI = false;
	static bool isGuidUI = false;
	static bool hasLoadedLevels = false;
	static bool isRush = false;
	static bool isBlackscreenProtection = false;
	static bool combatDisableReturn = false; //Prevents the combat system from exiting when an enemy leaves its territory mid-combo.
	static bool hasInitForTutorial = false; //When loading a fresh save, do some cool stuff.
	static float timer;
	static float timerStyle;
	static float holdTimer;
	static float stateTimer;
	static float tauntTimer;
	static float uiTimer;
	static float failSafeInitTimer;
	static float supremeQTETimer_CheckLaser;
	static float supremeQTETimer_PlayQTE;
	static float superRushScaling; //Placing here for easier access.
	static float letterBoxTimer;
	static int presentIndex = 0;
	static int supremeQTEChance = 1;
	static Sonic.StateID discardedState;
	public static List<string> Presents = new()
	{ "Smash", "ChargeAttack", "CrossSlash", "Cyloop", "SonicBoom", "Crasher", "AutoCombo", "HomingShot", "SpinSlash", "RecoverySmash", "AirTrick", "Stomping", "LoopKick", "QuickCyloop" };
	public static List<string> tPresents = new()
	{
		"AcceleLevel", "QuickCyloop"
	};
	public static void ReleaseSkill()
	{
		Console.WriteLine(tPresents[presentIndex]);
		Lua.Call("ReleasePresentSkill", tPresents[presentIndex]);
		presentIndex += 1;
		if (presentIndex > tPresents.Count - 1)
			presentIndex = 0;
	}
//
var SonicParams = Reflection.GetDataInfo<SonicParameters.Root>("player_common");
var AmyParams = Reflection.GetDataInfo<AmyParameters.Root>("amy_common");
var KnucklesParams = Reflection.GetDataInfo<KnucklesParameters.Root>("knuckles_common");
var TailsParams = Reflection.GetDataInfo<TailsParameters.Root>("tails_common");
var HitstopParam = Reflection.GetDataInfo<GameHitStopParameter.Root>("hitstop");
{
	/*		Initialize some vars 	*/
	if (Lua.GetState() == 0)
	{
	    return;
	}
	if (IS_WORLD_FLAG(IsCyberSpace))
	{
		return;
	}
	if (!hasLoadedLevels) //Not sure if this is the best solution, but I don't want to constantly make these function calls.
	{
		hasLoadedLevels = true;
		//Lua.Call("LoadLevel", "sound");
		Lua.Call("LoadLevel", "effectcommon_friends");
		Lua.Call("LoadLevel", "amyP");
		Lua.Call("LoadLevel", "tailsP");
		Lua.Call("LoadLevel", "knucklesP");
		//Lua.Call("LoadLevel", "minigame_sound");
	}
	var characterCheck = Player.GetPlayerType();
	AssignCharacter(characterCheck); //Updates "playerName" based on your current character.
	var curChar = CharacterList[playerName];
	curChar.ToggleAspect();
	if (failSafeInitTimer > 0.0f)
	{
		failSafeInitTimer -= Time.GetDeltaTime();
		if (failSafeInitTimer <= 0.0f)
		{
			failSafeInitTimer = 0.0f;
			Scoring.InitVars();
			curChar.InitCharacter();
		}
	}
	if (!IS_WORLD_FLAG(IsBattle) || IS_WORLD_FLAG(IsDead))
	{
		string exitReaction = "DEFAULT";
		string checkAnim = (Lua.Call<string>("GetCurrentAnimationName") != null) ? Lua.Call<string>("GetCurrentAnimationName") : "STAND";
		if (ReactionIndex.ContainsKey(checkAnim))
		{
			exitReaction = ReactionIndex[checkAnim];
			combatDisableReturn = false;
		}
		else if (IS_WORLD_FLAG(IsDead))
		{
			exitReaction = "COMBAT_EXIT";
			combatDisableReturn = false;
		}
		if (hasInit && exitReaction == "COMBAT_EXIT")
		{
			Console.WriteLine("INIT CALLED!");
			foreach (var Val in CharacterList.Values)
			{
				Val.hasInitChar = false;
			}
			if (hasChangedRush)
			{
				Aspects.SetRushDrain(7f, 14f);
				hasChangedRush = false;
			}
			SkillLink.ComboBonus.Clear();
			SkillLink.ChainLevel = 1;
			SkillLink.ChainActive = false;
			Scoring.InitVars();
			isStateDiscarded = false;
			isRush = false;
			Player.State.Restore(discardedState);
			if (uiTimer > 0.0f || isGuidUI)
				Utilities.UIToggle("GuidUI", "reset");
			if (isRankUI)
				Utilities.UIToggle("QuestTarget", "reset");
			if (letterBoxTimer > 0.0f)
				Utilities.UIToggle("LetterBox", "reset");
			timer = 0.0f;
			timerStyle = 0.0f;
			holdTimer = 0.0f;
			stateTimer = 0.0f;
			tauntTimer = 0.0f;
			uiTimer = 0.0f;
			letterBoxTimer = 0.0f;
			supremeQTETimer_CheckLaser = 0.0f;
			supremeQTETimer_PlayQTE = 0.0f;
			supremeQTEChance = 1;
			if (Aspects.aspectBonusCrit != 0 || Aspects.aspectBonusCritDamage != 0) //Another hacky fix for Cyberspace.
				curChar.ResetByDamage();
			hasInit = false;
		}
		if (!combatDisableReturn)
			return;
	} else
	{
		combatDisableReturn = true;
		hasInit = true;
	}
	if (Lua.Call<int>("GetValue", "Tutorial", 0) <= 3 && Lua.Call<int>("GetValue", "Tutorial", 0) > 1)
	{
		Lua.Call("SetHUDEnabled", "MainMenu", true);
		Lua.Call("SetHUDEnabled", "MapMenu", true);
		Lua.Call("SetPlayerAbilityEnabled", "ComboAttack", true);
		Lua.Call("SetPlayerAbilityEnabled", "Parry", true);
		Lua.Call("SetPlayerAbilityEnabled", "Lockon", true);
		if (!hasInitForTutorial)
		{
			bool setupStartingBonus = false;
			if (Lua.Call<int>("GetValue", "Tutorial", 0) <= 3 && !setupStartingBonus)
			{
				if (Lua.Call<int>("GetPlayerItemInfo", "ExpPoint") <= 130)
					Lua.Call("PlayerGetItem", "ExpPoint", 10000);
				Lua.Call("ReleasePresentSkill", "AcceleLevel");
				hasInitForTutorial = true;
				setupStartingBonus = true;
			}
			else if (Lua.Call<int>("GetValue", "Tutorial", 0) > 3)
				hasInitForTutorial = true;
		}
	}
	if (playerName == "Super")
	{
		var checkState = Player.State.GetCurrentStateID<Sonic.StateID>();
		if (checkState == Sonic.StateID.StateHoldStand || checkState == Sonic.StateID.StateCaught) //Pauses combo behavior during cutscenes.
		{
			if (isRush)
			{
				Lua.Call("SetPhantomRushGauge", Bonuses.pauseVal); //Keeps Rush consistent during Cyloop animations.
			}
			isBlackscreenProtection = true;
			return;
		}
	}
	/* 		Timer logic 		*/
	if (currentAnim == "")
		currentAnim = Lua.Call<string>("GetCurrentAnimationName");
	timer += Time.GetDeltaTime();
	timerStyle += Time.GetDeltaTime();
	if (isBlackscreenProtection)
	{
		isBlackscreenProtection = false;
		Lua.Call("FadeIn", 0.05f);
	}
	if (holdTimer > 0.0f) //holdTimer is set in the function HoldOnOff, which is part of CharacterBase.
	{
		holdTimer -= Time.GetDeltaTime();
		if (holdTimer <= 0.0f)
		{
			holdTimer = 0.0f;
			Utilities.HoldOnOff(false);
		}
	}
	if (uiTimer > 0.0f)
	{
		uiTimer -= Time.GetDeltaTime();
		if (uiTimer <= 0.0f)
		{
			Console.WriteLine("GUID_UI OFF!!!!!!!!");
			uiTimer = 0.0f;
			Utilities.UIToggle("GuidUI", "reset");
		}
	}
	if (letterBoxTimer > 0.0f)
	{
		letterBoxTimer -= Time.GetDeltaTime();
		if (letterBoxTimer <= 0.0f)
		{
			letterBoxTimer = 0.0f;
			Utilities.UIToggle("LetterBox", "reset");
		}
	}
	if (stateTimer > 0.0f)
	{
		stateTimer -= Time.GetDeltaTime();
		if (stateTimer <= 0.0f)
		{
			stateTimer = 0.0f;
			Player.State.Restore(discardedState);
			isStateDiscarded = false;
		}
	}
	incrementTaunt(curChar);
	if (timerStyle >= 0.5f)
	{
		if (Scoring.styleScore >= 45)
		{
			if (Scoring.comboLockout > 0)
				Scoring.comboLockout -= 1;
			else
			{
				Scoring.styleScore -= Math.Max(1, Scoring.comboRank);
				Scoring.StyleRankCall(curChar.myRanks);
			}
		}
		timerStyle = 0f;
	}
	if (Bonuses.bonusOn)
	{
		Bonuses.bonusDuration -= Time.GetDeltaTime();
		if (Bonuses.bonusDuration <= 0.0f)
		{
			Bonuses.SetupBonus("Reset");
		}
	}
	if (Bonuses.isPause)
	{
		float rushGauge = Utilities.GetBattleInformation("PhantomRush");
		if (rushGauge < Bonuses.pauseVal)
			Lua.Call("SetPhantomRushGauge", Bonuses.pauseVal);
	}
	if (timer >= 0.2f)
	{
		//Console.WriteLine(Player.State.GetCurrentStateID<Sonic.StateID>());
		Aspects.SetCritByTension();
		curChar.UpdateCyloopBehavior(); //The other characters handle Cyloop differently, so I want some extra control over this
		if (currentAnim != Lua.Call<string>("GetCurrentAnimationName"))
		{
			lastAnim = currentAnim;
			currentAnim = Lua.Call<string>("GetCurrentAnimationName");
			//Console.WriteLine(currentAnim);
			if (MoveIndex.ContainsKey(currentAnim))
			{
				int styleBon = Scoring.UpdateStale(currentAnim, ref curChar.Moves);
				curChar.OnStyleUpdate(styleBon);
			}
			if (ReactionIndex.ContainsKey(currentAnim))
			{
				string reaction = ReactionIndex[currentAnim];
				switch (reaction)
				{
					case "Damage": case "HeavyKB":
						curChar.ResetByDamage();
						Scoring.styleScore = (int)Math.Floor(Scoring.styleScore/2f);
						if (playerName == "Super")
						{
							Bonuses.isPause = false;
							Bonuses.pauseVal = 0f;
						}
						break;
				}
			}
		};
		timer = 0f;
		currentAnim = "";
	}
	/* Super Sonic warnings/Phantom Rush behavior */
	if (playerName == "Super")
	{
		if (!curChar.hasInitChar)
		{
			curChar.hasInitChar = true;
			curChar.InitCharacter(); //HMM force-closes the game when trying to boot a code that runs methods in a class constructor.
		}
		if (curChar.warningList["ringWarning"] == false) //Yes, I specifically want to check for false on these warnings.
		{
			int ringNum = Lua.Call<int>("GetPlayerItemInfo", "Ring");
			if (ringNum <= 50)
			{
				curChar.warningList["ringWarning"] = true;
				Utilities.UIToggle("HeaderWindow", "enable", "At this rate, Super Sonic will run out of rings!\nTo replenish your rings, use a punch\nor kick as the first move in a Skill Link.\n(REMINDER: Skill Links are initiated by a successful Parry.)", "Replenish Rings");
			}
		}
		Utilities.IsRushActive();
		if (!isRush)
		{
			curChar.hasSetRush = false;
			Bonuses.isPause = false;
			Bonuses.pauseVal = 0f;
			float incVal = 1f + Utilities.ToDecimal(Lua.Call<int>("GetPowerLevel")/superRushScaling, 2);
			float upVal = Utilities.GetBattleInformation("PhantomRush") + ((incVal * Math.Max(1, ((Scoring.comboRank/2) * 1.5f)))/45f) * Scoring.rushMultiplier);
			//Console.WriteLine("UPVAL: " + upVal);
			Lua.Call("SetPhantomRushGauge", upVal);
		} 
		else
		{
			if (!curChar.hasSetRush)
			{
				curChar.hasSetRush = true;
				Bonuses.isPause = true;
				Bonuses.pauseVal = Utilities.GetBattleInformation("PhantomRush");
				if (curChar.warningList["rushWarning"] == false)
				{
					curChar.warningList["rushWarning"] = true;
					Utilities.UIToggle("HeaderWindow", "enable", "Phantom Rush works differently for Super Sonic.\nOnce the meter is filled, it will not deplete over time,\nbut every attack will drain it.\n\nThe meter will passively regenerate over time based on your Combo Rank.", "Super Phantom Rush!!!");
				}
			}
		}
		if (supremeQTETimer_CheckLaser > 0.0f) //Just makes sure we don't try running the DiEvent stuff multiple times per laser.
		{
			supremeQTETimer_CheckLaser -= Time.GetDeltaTime();
			if (supremeQTETimer_CheckLaser <= 0.0f)
			{
				supremeQTETimer_CheckLaser = 0.0f;
			}
		}
		if (supremeQTETimer_PlayQTE > 0.0f)
		{
			supremeQTETimer_PlayQTE -= Time.GetDeltaTime();
			if (supremeQTETimer_PlayQTE <= 0.0f)
			{
				supremeQTETimer_PlayQTE = 0.0f;
				Lua.Call("PlayDiEvent", "test_bo2115");
			}
		}
		if (currentAnim == "LASER_START" && supremeQTETimer_CheckLaser <= 0.0f)
		{
			supremeQTETimer_CheckLaser = 4f;
			int playQTE = qteRNG.Next(supremeQTEChance, 5);
			if (playQTE == 4)
			{
				supremeQTEChance = 1;
				supremeQTETimer_PlayQTE = 0.8f;
			}
			else
			{
				supremeQTEChance += 1;
				if (supremeQTEChance > 4)
					supremeQTEChance = 4; //Redundant? Probably. Am I taking chances? Nope. Sanity checky, baybeeeee
				//Console.WriteLine("QTE MISSED. CHANCE: " + supremeQTEChance + " RNG WAS: " + playQTE);
			}
		}
	}
}
	//Utilities.IsRushActive(); 
Patch "Fix Giganto Scripting"
#lib "Lua"
{
	Lua.CreateLineHook
    (
        """
            NextSequence(40) -- we're replacing code here and our query removes this from a previous function

            FadeIn(0)
        """,
        
        "w1r03_sequence35.lua", "NextSequence(40)\n  Exit()",
        
        HookBehavior.Replace
    );
}
Library "GameHitStopParameter"
{
    #load "System.Numerics.dll"

    using System.Numerics;
    using System.Runtime.InteropServices;

    [StructLayout(LayoutKind.Explicit, Size = 8)]
    public struct UnmanagedString
    {
        [FieldOffset(0)] public long pValue;

        public string Value
        {
            get
            {
                if (pValue == 0)
                    return string.Empty;

                return Marshal.PtrToStringAnsi((nint)pValue);
            }

            set => pValue = (long)Marshal.StringToHGlobalAnsi(value);
        }

        public UnmanagedString(string in_value)
        {
            Value = in_value;
        }

        public static implicit operator UnmanagedString(string in_value)
        {
            return new UnmanagedString(in_value);
        }

        public static bool operator ==(UnmanagedString in_left, string in_right)
        {
            return in_left.Value == in_right;
        }

        public static bool operator !=(UnmanagedString in_left, string in_right)
        {
            return !(in_left == in_right);
        }

        public override bool Equals(object in_obj)
        {
            if (in_obj is string str)
                return Value == str;

            return base.Equals(in_obj);
        }

        public override int GetHashCode()
        {
            return Value.GetHashCode();
        }

        public override string ToString()
        {
            return Value;
        }
    }

    [StructLayout(LayoutKind.Explicit, Size = 0x28)]
    public struct GameHitStopParameterData
    {
        [FieldOffset(0x00)] public UnmanagedString name;
        [FieldOffset(0x10)] public float scale;
        [FieldOffset(0x14)] public float time;
        [FieldOffset(0x18)] public float easeOutTime;
        [FieldOffset(0x1C)] public float delayTime;
        [FieldOffset(0x20)] public bool layerPlayer;
        [FieldOffset(0x21)] public bool layerEnemy;
        [FieldOffset(0x22)] public bool layerDamagedEnemy;
        [FieldOffset(0x23)] public bool layerCamera;
        [FieldOffset(0x24)] public bool layerOthers;
    }

    [StructLayout(LayoutKind.Explicit, Size = 0xA00)]
    public struct Root
    {
        [FieldOffset(0x00)] public unsafe fixed byte /* GameHitStopParameterData[64] */ _data[2560];

        public unsafe GameHitStopParameterData* data
        {
            get
            {
                fixed (byte* p_data = _data)
                    return (GameHitStopParameterData*)p_data;
            }
        }
    }
}  

